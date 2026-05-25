# Powerset Research Data

This repo documents how to access the data backing [Powerset Research](https://research.powerset.co/), either directly or using an AI agent via MCP.

## GitHub Data

Public dataset of ~400,000 active GitHub repositories maintained by [Powerset](https://powerset.co). We use this data internally to identify top open source developers, diligence fast-growing repos, and understand technology trends. Now the same data is freely available for your agents and SQL workflows.

![Powerset Research Data](./powerset-mcp.gif)

The dataset includes repositories, contributors, activity, stars, languages, categories, README summaries, embeddings, and project metadata. No credentials are required.

Example questions you can ask:

- What are the fastest-growing terminal coding agents?
- Find the 5 most impressive systems architects in San Francisco.
- What are the most popular AI repos with heavy Bun dependency?

### Access methods

There are two primary ways to use this data:

1. **MCP server** - connect Claude, Codex, Cursor, or another MCP-compatible client and ask questions conversationally.
2. **DuckDB + Agent Skills** - attach directly to the frozen DuckLake catalog and run SQL yourself, optionally giving your agent the included skill for schema context, query patterns, and examples.

### MCP server

Use MCP if you want an agent to explore the data conversationally.

Endpoint:

```text
https://research-mcp.powerset.dev/mcp/
```

No authentication is required. The server exposes tools to run SQL against the public DuckLake and inspect the schema.

#### Claude Code

```bash
claude mcp add --transport streamable-http powerset-research https://research-mcp.powerset.dev/mcp/
```

#### OpenAI Codex

```bash
codex mcp add powerset-research --url https://research-mcp.powerset.dev/mcp/
```

#### Claude Desktop

Add this to your Claude Desktop config (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "powerset-research": {
      "type": "http",
      "url": "https://research-mcp.powerset.dev/mcp/"
    }
  }
}
```

#### Cursor

Add this to your Cursor MCP settings (`.cursor/mcp.json` in your project or global config):

```json
{
  "mcpServers": {
    "powerset-research": {
      "type": "http",
      "url": "https://research-mcp.powerset.dev/mcp/"
    }
  }
}
```

### DuckDB + Agent Skills

Use DuckDB if you want direct SQL access. The data is published as a [frozen DuckLake](https://ducklake.select/2025/10/24/frozen-ducklake/) catalog backed by Parquet files on Cloudflare R2.

If your agent can run DuckDB locally, you can also install the included [powerset-research-data skill](powerset-research-data/SKILL.md). The skill gives the agent schema context, query guidelines, and examples for working with the dataset directly through DuckDB.

#### DuckDB setup

Run this once per DuckDB session:

```sql
INSTALL ducklake;
INSTALL httpfs;
LOAD ducklake;
LOAD httpfs;

ATTACH 'ducklake:https://research-data.powerset.dev/github-public/latest/public.ducklake' AS github (READ_ONLY);
```

After attaching, reference tables as `github.<table>`.

#### CLI setup

You can also use the DuckDB CLI directly:

```bash
duckdb -c "
  INSTALL ducklake; INSTALL httpfs; LOAD ducklake; LOAD httpfs;
  ATTACH 'ducklake:https://research-data.powerset.dev/github-public/latest/public.ducklake' AS github (READ_ONLY);
  SELECT count(*) FROM github.repos;
"
```

#### Example queries

Repos with the most stars:

```sql
SELECT name_with_owner, stars_count, pushed_at
FROM github.repos
ORDER BY stars_count DESC
LIMIT 20;
```

Top contributors to a repository:

```sql
SELECT login, contributions
FROM github.repo_contributors
WHERE repo_node_id = (
    SELECT repo_node_id
    FROM github.repos
    WHERE name_with_owner = 'duckdb/duckdb'
)
ORDER BY contributions DESC
LIMIT 10;
```

Recent pull request activity for a repository:

```sql
SELECT pull_number, title, state, user_login, created_at, merged_at
FROM github.repo_pulls
WHERE repo_node_id = (
    SELECT repo_node_id
    FROM github.repos
    WHERE name_with_owner = 'duckdb/duckdb'
)
ORDER BY created_at DESC
LIMIT 20;
```

Daily star history for a repository:

```sql
SELECT starred_date, stars_delta
FROM github.repo_stars_daily
WHERE repo_node_id = (
    SELECT repo_node_id
    FROM github.repos
    WHERE name_with_owner = 'duckdb/duckdb'
)
ORDER BY starred_date DESC
LIMIT 30;
```

Repos by category:

```sql
SELECT r.name_with_owner, r.stars_count, c.top_category, c.similarity
FROM github.repo_categories c
JOIN github.repos r USING (repo_node_id)
WHERE c.top_category = 'AI & Machine Learning'
ORDER BY r.stars_count DESC
LIMIT 20;
```

Issue volume by repo for an organization:

```sql
SELECT r.name_with_owner, COUNT(*) AS issue_count
FROM github.repo_issues i
JOIN github.repos r USING (repo_node_id)
WHERE r.name_with_owner LIKE 'vercel/%'
  AND i.is_pull_request = false
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;
```

Find repositories similar to `duckdb/duckdb` using README summary embeddings, limited to popular candidates:

```sql
WITH anchor AS MATERIALIZED (
    SELECT e.repo_node_id, e.embedding
    FROM github.repo_readme_summary_embeddings e
    JOIN github.repos r USING (repo_node_id)
    WHERE r.name_with_owner = 'duckdb/duckdb'
      AND e.embedding IS NOT NULL
), candidate_ids AS MATERIALIZED (
    SELECT repo_node_id, name_with_owner
    FROM github.repos
    WHERE stars_count >= 1000
      AND name_with_owner <> 'duckdb/duckdb'
), candidates AS (
    SELECT c.name_with_owner, e.embedding
    FROM candidate_ids c
    JOIN github.repo_readme_summary_embeddings e USING (repo_node_id)
    WHERE e.embedding IS NOT NULL
)
SELECT candidates.name_with_owner,
       list_cosine_similarity(anchor.embedding, candidates.embedding) AS similarity
FROM anchor
CROSS JOIN candidates
ORDER BY similarity DESC
LIMIT 10;
```

Filter candidates before joining embeddings, as shown above. The embedding table is large, so comparing against the full table can be slow and memory-intensive.

### Tables

The `github` catalog contains the following tables. Repo tables join on `repo_node_id`. User data joins via `user_id`, for example `repo_contributors.user_id` to `github_users.user_id`.

| Table                            | Description                                                                                                                     | Key columns                                                                                                             |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `repos`                          | Active repositories with at least 10 GitHub stars. Use `stars_count` for current star totals.                                   | `repo_node_id`, `name_with_owner`, `stars_count`, `fork_count`, `pushed_at`, `created_at`                               |
| `repo_metadata`                  | Repository metadata: description, topics, language, license, owner type, watchers, open issue counts, feature flags.            | `repo_node_id`, `description`, `topics`, `language`, `license_key`, `owner_type`, `watchers_count`, `open_issues_count` |
| `repo_scores`                    | Powerset-computed ranking scores and cohort groupings.                                                                          | `repo_node_id`, `name_with_owner`, `score_overall`, `cohort_group`, `star_cohort`                                       |
| `repo_categories`                | Top category assignment per repository.                                                                                         | `repo_node_id`, `top_category_id`, `top_category`, `similarity`                                                         |
| `repo_category_similarities`     | Repository-to-category similarity scores.                                                                                       | `repo_node_id`, `category_id`, `similarity`                                                                             |
| `repo_contributors`              | Contributor information and contribution counts per repository.                                                                 | `repo_node_id`, `user_id`, `login`, `contributions`, `type`                                                             |
| `repo_readme_summaries`          | Generated plain-text summaries of repository README content.                                                                    | `repo_node_id`, `name_with_owner`, `summary`, `content_hash`, `generated_at`, `tier`                                    |
| `repo_readme_summary_embeddings` | README summary embeddings (`FLOAT[]`) for semantic similarity search.                                                           | `repo_node_id`, `content_hash`, `embedding`, `_generated_at`                                                            |
| `github_users`                   | Public GitHub user profile fields for users in the corpus.                                                                      | `user_id`, `login`, `name`, `company`, `location`, `bio`, `followers_count`                                             |
| `repo_pulls`                     | Pull request metadata from roughly the last two years. Body text is excluded.                                                   | `repo_node_id`, `pull_number`, `title`, `state`, `user_id`, `user_login`, `created_at`, `merged_at`                     |
| `repo_issues`                    | Issue metadata from roughly the last two years. Body text is excluded. Includes pull requests; use `is_pull_request` to filter. | `repo_node_id`, `issue_number`, `title`, `state`, `user_id`, `user_login`, `created_at`, `is_pull_request`              |
| `repo_stars_daily`               | Daily star counts per repository.                                                                                               | `repo_node_id`, `starred_date`, `stars_delta`                                                                           |

### Data coverage and limitations

- **Repository universe**: active public repositories with at least 10 GitHub stars that have been pushed within the last 90 days.
- **Pull requests**: only PRs created within the last ~2 years are present. Older PRs are not included even if recently updated or merged.
- **Issues**: only issues created within the last ~2 years are present. Older issues are not included even if recently updated or closed. The GitHub Issues API returns both issues and pull requests; use the `is_pull_request` column to distinguish them.
- **Star history**: the GitHub Stargazers API returns at most ~40,000 individual star events per repository. For repos with more than 40k stars, `repo_stars_daily` has incomplete history and `SUM(stars_delta)` will undercount the true total. Use `repos.stars_count` for accurate current star counts.
- **Snapshots**: the data is published as immutable timestamped snapshots. The `latest/` pointer is updated when a new snapshot is published. Snapshots are not real-time; there is a delay between GitHub activity and data availability.
