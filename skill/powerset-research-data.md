# Powerset Research Data — DuckDB Query Skill

You have access to a public GitHub dataset maintained by [Powerset](https://powerset.co) via a frozen DuckLake catalog. Use DuckDB to query it directly — no credentials or API keys required.

## Setup

Run this once per session to attach the catalog:

```sql
INSTALL ducklake;
INSTALL httpfs;
LOAD ducklake;
LOAD httpfs;

ATTACH 'ducklake:https://research-data.powerset.dev/github-public/latest/public.ducklake' AS github (READ_ONLY);
```

After attaching, reference tables as `github.<table>`.

## Discovering the schema

List all tables:

```sql
SHOW TABLES FROM github;
```

Show columns for a specific table:

```sql
DESCRIBE github.repos;
```

Show columns for all tables at once:

```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_catalog = 'github'
ORDER BY table_name, ordinal_position;
```

## Tables

| Table | Description | Key columns |
|-------|-------------|-------------|
| `repos` | Core repository universe (~400k active repos with ≥10 stars) | `repo_node_id`, `full_name`, `language`, `stars_count`, `forks_count`, `pushed_at`, `created_at` |
| `repo_metadata` | Extended attributes: description, topics, license, owner type, feature flags | `repo_node_id`, `language`, `owner_type`, `visibility`, `topics` |
| `repo_scores` | Powerset-computed ranking scores | `repo_node_id`, `score_overall`, `cohort_group`, `star_cohort` |
| `repo_categories` | Top category assignment per repo | `repo_node_id`, `top_category`, `top_category_id`, `similarity` |
| `repo_category_similarities` | Full repo-to-category similarity matrix | `repo_node_id`, `category_id`, `similarity` |
| `repo_contributors` | Contributor list with commit counts | `repo_node_id`, `user_id`, `login`, `contributions` |
| `repo_readme_summaries` | LLM-generated README summaries | `repo_node_id`, `summary` |
| `repo_readme_summary_embeddings` | Embeddings for semantic similarity search | `repo_node_id`, `embedding` (FLOAT[]) |
| `github_users` | GitHub user profiles | `user_id`, `login`, `company`, `location`, `bio` |
| `repo_pulls` | Pull request metadata (body text excluded) | `repo_node_id`, `pull_number`, `title`, `state`, `user_login`, `created_at`, `merged_at` |
| `repo_issues` | Issue metadata (body text excluded) | `repo_node_id`, `issue_number`, `title`, `state`, `user_login`, `created_at`, `is_pull_request` |
| `repo_stars_daily` | Daily star counts | `repo_node_id`, `starred_date`, `stars_delta` |

**Join keys**: repo tables join on `repo_node_id`. User data joins via `user_id` (e.g., `repo_contributors.user_id` to `github_users.user_id`).

## Data coverage

- **Repository universe**: active public repos with ≥10 stars and a push within the last 90 days.
- **Pull requests and issues**: only records created within the last ~2 years. Older records are not present.
- **Issues include PRs**: the GitHub Issues API returns both. Use `is_pull_request` to filter.
- **Star history**: capped at ~40,000 events per repo (GitHub API limit). `SUM(stars_delta)` undercounts for popular repos. Use `repos.stars_count` for accurate totals.
- **Snapshots**: data is published as immutable snapshots. Not real-time.

## Query guidelines

- Always use `LIMIT` when exploring. Start with `LIMIT 20` and increase as needed.
- Prefer aggregations (`COUNT`, `SUM`, `GROUP BY`) over fetching raw rows.
- Do not `SELECT *` on `repo_readme_summary_embeddings` — the embedding column is large. Select specific columns instead.
- Use `repo_readme_summaries` (not raw READMEs) to understand what a project does.
- Filter by `repo_node_id` when querying activity tables (`repo_pulls`, `repo_issues`, `repo_stars_daily`) for specific repos.

## Example queries

Top repos by stars:

```sql
SELECT full_name, language, stars_count, pushed_at
FROM github.repos
ORDER BY stars_count DESC
LIMIT 20;
```

Contributors to a specific repo:

```sql
SELECT login, contributions
FROM github.repo_contributors
WHERE repo_node_id = (
    SELECT repo_node_id FROM github.repos WHERE full_name = 'duckdb/duckdb'
)
ORDER BY contributions DESC
LIMIT 10;
```

Recent PR activity:

```sql
SELECT pull_number, title, state, user_login, created_at
FROM github.repo_pulls
WHERE repo_node_id = (
    SELECT repo_node_id FROM github.repos WHERE full_name = 'duckdb/duckdb'
)
ORDER BY created_at DESC
LIMIT 20;
```

Daily star history:

```sql
SELECT starred_date, stars_delta
FROM github.repo_stars_daily
WHERE repo_node_id = (
    SELECT repo_node_id FROM github.repos WHERE full_name = 'duckdb/duckdb'
)
ORDER BY starred_date DESC
LIMIT 30;
```

Find similar repos using README embeddings:

```sql
WITH anchor AS (
    SELECT repo_node_id, embedding
    FROM github.repo_readme_summary_embeddings
    WHERE repo_node_id = (
        SELECT repo_node_id FROM github.repos WHERE full_name = 'duckdb/duckdb'
    )
      AND embedding IS NOT NULL
)
SELECT r.full_name,
       list_cosine_similarity(anchor.embedding, b.embedding) AS similarity
FROM anchor
JOIN github.repo_readme_summary_embeddings b
  ON b.repo_node_id <> anchor.repo_node_id
  AND b.embedding IS NOT NULL
JOIN github.repos r ON r.repo_node_id = b.repo_node_id
ORDER BY similarity DESC
LIMIT 10;
```

Repos by category:

```sql
SELECT r.full_name, r.stars_count, c.top_category, c.similarity
FROM github.repo_categories c
JOIN github.repos r USING (repo_node_id)
WHERE c.top_category = 'Machine Learning'
ORDER BY r.stars_count DESC
LIMIT 20;
```

Issue volume by repo for an org:

```sql
SELECT r.full_name, COUNT(*) AS issue_count
FROM github.repo_issues i
JOIN github.repos r USING (repo_node_id)
WHERE r.full_name LIKE 'vercel/%'
  AND i.is_pull_request = false
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;
```
