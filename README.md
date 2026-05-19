# Powerset Research Data

Public dataset of GitHub repository metadata, activity, scores, and contributor information maintained by [Powerset](https://powerset.co). The data covers active public repositories with at least 10 GitHub stars.

## Access methods

There are three ways to use this data:

1. **DuckDB** — attach the frozen DuckLake catalog directly and run SQL locally
2. **MCP server** — connect an AI assistant (Claude, Codex, etc.) to the hosted MCP endpoint
3. **Agent skill** — drop the [skill file](skill/powerset-research-data.md) into your agent's context to give it the schema, query patterns, and best practices for working with the data via DuckDB

## DuckDB (local)

The data is published as a [frozen DuckLake](https://duckdb.org/2025/05/16/ducklake.html) catalog backed by Parquet files on Cloudflare R2. You can attach it from any DuckDB client with no credentials required.

### Setup

```sql
INSTALL ducklake;
INSTALL httpfs;
LOAD ducklake;
LOAD httpfs;

ATTACH 'ducklake:https://research-data.powerset.dev/github-public/latest/public.ducklake' AS github (READ_ONLY);
```

### Example queries

Repos with the most stars:

```sql
SELECT full_name, language, stars_count, pushed_at
FROM github.repos
ORDER BY stars_count DESC
LIMIT 20;
```

Top contributors to a repository:

```sql
SELECT login, contributions
FROM github.repo_contributors
WHERE repo_node_id = (
    SELECT repo_node_id FROM github.repos
    WHERE full_name = 'duckdb/duckdb'
)
ORDER BY contributions DESC
LIMIT 10;
```

Recent pull request activity for a repository:

```sql
SELECT pull_number, title, state, user_login, created_at, merged_at
FROM github.repo_pulls
WHERE repo_node_id = (
    SELECT repo_node_id FROM github.repos
    WHERE full_name = 'duckdb/duckdb'
)
ORDER BY created_at DESC
LIMIT 20;
```

Daily star history for a repository:

```sql
SELECT starred_date, stars_delta
FROM github.repo_stars_daily
WHERE repo_node_id = (
    SELECT repo_node_id FROM github.repos
    WHERE full_name = 'duckdb/duckdb'
)
ORDER BY starred_date DESC
LIMIT 30;
```

Find similar repositories using README embeddings:

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

### CLI

You can also use the DuckDB CLI directly:

```bash
duckdb -c "
  INSTALL ducklake; INSTALL httpfs; LOAD ducklake; LOAD httpfs;
  ATTACH 'ducklake:https://research-data.powerset.dev/github-public/latest/public.ducklake' AS github (READ_ONLY);
  SELECT count(*) FROM github.repos;
"
```

## MCP server

An MCP server is available at:

```
https://research-mcp.powerset.dev/mcp/
```

No authentication is required. The server exposes two tools:

- `query_sql` — run a read-only SQL query against the public DuckLake
- `get_schema` — list available tables and suggested filters

### Claude Code

```bash
claude mcp add --transport streamable-http powerset-research https://research-mcp.powerset.dev/mcp/
```

### Claude Desktop

Add to your Claude Desktop config (`claude_desktop_config.json`):

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

### OpenAI Codex

```bash
codex mcp add powerset-research --url https://research-mcp.powerset.dev/mcp/
```

### Cursor

Add to your Cursor MCP settings (`.cursor/mcp.json` in your project or global config):

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

## Tables

The `github` catalog contains the following tables. Repo tables join on `repo_node_id`. User data joins via `user_id` (e.g., `repo_contributors.user_id` to `github_users.user_id`).

| Table | Description |
|-------|-------------|
| `repos` | Active repositories with at least 10 GitHub stars. Use `stars_count` for current star totals. |
| `repo_metadata` | Repository metadata: description, topics, language, license, owner type, watchers, open issue counts. |
| `repo_scores` | Ranked repository scores used by Powerset Research. |
| `repo_categories` | Top category assignment per repository. |
| `repo_category_similarities` | Repository-to-category similarity scores. |
| `repo_contributors` | Contributor information and contribution counts per repository. |
| `repo_readme_summaries` | Generated summaries of repository README content. |
| `repo_readme_summary_embeddings` | README summary embeddings (`FLOAT[]`) for similarity search. |
| `github_users` | Public GitHub user profile fields for users in the corpus. |
| `repo_pulls` | Pull request metadata (body text excluded). |
| `repo_issues` | Issue metadata (body text excluded). Includes pull requests; use `is_pull_request` to filter. |
| `repo_stars_daily` | Daily star counts per repository: `repo_node_id`, `starred_date`, `stars_delta`. |

## Data coverage and limitations

- **Repository universe**: active public repositories with at least 10 GitHub stars that have been pushed to within the last 90 days.
- **Pull requests**: only PRs created within the last ~2 years are present. Older PRs are not included even if recently updated or merged.
- **Issues**: only issues created within the last ~2 years are present. Older issues are not included even if recently updated or closed. The GitHub Issues API returns both issues and pull requests; use the `is_pull_request` column to distinguish them.
- **Star history**: the GitHub Stargazers API returns at most ~40,000 individual star events per repository. For repos with more than 40k stars, `repo_stars_daily` has incomplete history and `SUM(stars_delta)` will undercount the true total. Use `repos.stars_count` for accurate current star counts.
- **Snapshots**: the data is published as immutable timestamped snapshots. The `latest/` pointer is updated when a new snapshot is published. Snapshots are not real-time; there is a delay between GitHub activity and data availability.

