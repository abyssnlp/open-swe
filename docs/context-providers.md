# Context Providers Guide

Open SWE agents work best when they have rich context about the task, the codebase, and the surrounding systems. Out of the box, Open SWE provides context from GitHub, Linear, and Slack. This guide shows how to connect additional enterprise data sources — Jira, Confluence, Postgres, Snowflake, AWS MWAA (Airflow), knowledge bases (vector DBs), and more — so the agent can pull in the information it needs to complete tasks.

> **Quick start:** Each context source is added as a [tool](../CUSTOMIZATION.md#3-tools). Create a function, register it in `agent/server.py`, and the agent will use it when it needs context from that source.

---

## How context flows into the agent

Open SWE uses three layers of context:

```
┌──────────────────────────────────────────────────────────────┐
│  1. System prompt (static)                                   │
│     • AGENTS.md from the repo                                │
│     • default_prompt.md (org-level conventions)              │
│     • Prompt sections (coding standards, workflow, etc.)     │
└──────────────────────────────────────────────────────────────┘
                          ▼
┌──────────────────────────────────────────────────────────────┐
│  2. Trigger context (per-task)                               │
│     • Linear issue title + description + comments            │
│     • Slack thread history                                   │
│     • GitHub issue/PR body + comments                        │
│     • Jira ticket details (if you add a Jira trigger)        │
└──────────────────────────────────────────────────────────────┘
                          ▼
┌──────────────────────────────────────────────────────────────┐
│  3. Tool-fetched context (on-demand)                         │
│     • Agent decides when to call each tool                   │
│     • Each tool returns structured data                      │
│     • GitHub files, Confluence pages, DB query results,      │
│       Airflow DAG status, knowledge base articles, etc.      │
└──────────────────────────────────────────────────────────────┘
```

**Layer 1** is configured once and applies to every task. See [CUSTOMIZATION.md § 5](../CUSTOMIZATION.md#5-system-prompt).

**Layer 2** is assembled automatically from the trigger event (the Linear issue, Slack thread, or GitHub comment that started the run).

**Layer 3** is where context providers come in. Each provider is a tool function that the agent calls when it decides it needs information from that source. The agent reads the tool's docstring to understand what it does and when to use it, then calls it with appropriate arguments.

### Design principles

When building context provider tools:

1. **Read-only by default.** Context tools should retrieve information, not modify external systems. Separate read and write into different tools.
2. **Narrowly scoped.** Each tool should do one thing well — don't build a single "query anything" tool.
3. **Structured output.** Return dictionaries or concise strings the agent can reason about. Avoid dumping raw HTML or massive JSON blobs.
4. **Explicit arguments.** Require the agent to specify what it's looking for (e.g. a Jira issue key, a Confluence page title) rather than accepting unconstrained queries.
5. **Limits and timeouts.** Cap result sizes and set timeouts to prevent runaway queries.
6. **Clear docstrings.** The tool docstring is the only description the agent sees. Write it like you're explaining the tool to a colleague.

---

## Built-in context: GitHub

Open SWE includes several built-in tools for GitHub context. These are registered by default in `agent/server.py`:

| Tool | Purpose |
|---|---|
| `list_repos` | List repositories in a GitHub org or user account |
| `get_branch_name` | Get the git branch name for the current thread |
| `commit_and_open_pr` | Commit changes and open/update a draft PR |
| `github_comment` | Post comments on issues or PRs |
| `list_pr_reviews` / `get_pr_review` | Read PR review details |
| `create_pr_review` / `update_pr_review` / `submit_pr_review` | Create and manage PR reviews |
| `list_pr_review_comments` | List review comments on a PR |

Additionally, the agent has full shell access in the sandbox, so it can run `git log`, `git blame`, `git diff`, `grep`, `find`, and any other CLI tool to explore the codebase.

---

## Adding context providers

Each context source follows the same pattern:

1. Create a tool file in `agent/tools/`
2. Implement a function with a clear docstring and typed parameters
3. Register it in `agent/server.py`

The sections below provide complete examples for each data source.

---

### Jira

Provide the agent with Jira issue details, linked issues, comments, and transitions.

#### Prerequisites

- Jira Cloud or Jira Server instance
- API token (Jira Cloud) or personal access token (Jira Server)
- `requests` (already available in Open SWE)

#### Environment variables

```bash
JIRA_BASE_URL="https://your-org.atlassian.net"
JIRA_USER_EMAIL="service-account@your-org.com"
JIRA_API_TOKEN="your-api-token"
```

#### Tool implementation

```python
# agent/tools/jira_context.py
import os
import requests
from typing import Any

JIRA_BASE_URL = os.environ.get("JIRA_BASE_URL", "")
JIRA_AUTH = (
    os.environ.get("JIRA_USER_EMAIL", ""),
    os.environ.get("JIRA_API_TOKEN", ""),
)


def jira_get_issue(issue_key: str) -> dict[str, Any]:
    """Get details of a Jira issue by its key (e.g. "ENG-1234").

    Returns the issue summary, description, status, priority, assignee,
    reporter, labels, components, and linked issues. Use this to understand
    ticket requirements and context before implementing changes.

    Args:
        issue_key: The Jira issue key (e.g. "ENG-1234", "PROJ-567")

    Returns:
        Dictionary with issue details
    """
    url = f"{JIRA_BASE_URL}/rest/api/3/issue/{issue_key}"
    params = {"fields": "summary,description,status,priority,assignee,reporter,labels,components,issuelinks,comment"}
    response = requests.get(url, auth=JIRA_AUTH, params=params, timeout=30)
    response.raise_for_status()
    data = response.json()
    fields = data["fields"]
    return {
        "key": data["key"],
        "summary": fields.get("summary"),
        "description": _extract_text(fields.get("description")),
        "status": fields.get("status", {}).get("name"),
        "priority": fields.get("priority", {}).get("name"),
        "assignee": fields.get("assignee", {}).get("displayName"),
        "reporter": fields.get("reporter", {}).get("displayName"),
        "labels": fields.get("labels", []),
        "components": [c["name"] for c in fields.get("components", [])],
        "linked_issues": [
            {
                "type": link.get("type", {}).get("name"),
                "key": (link.get("outwardIssue") or link.get("inwardIssue", {})).get("key"),
                "summary": (link.get("outwardIssue") or link.get("inwardIssue", {})).get("fields", {}).get("summary"),
            }
            for link in fields.get("issuelinks", [])
        ],
    }


def jira_get_comments(issue_key: str, max_results: int = 10) -> list[dict[str, Any]]:
    """Get comments on a Jira issue.

    Returns the most recent comments on the issue, ordered by creation date.
    Use this to understand discussion context and any clarifications.

    Args:
        issue_key: The Jira issue key (e.g. "ENG-1234")
        max_results: Maximum number of comments to return (default: 10)

    Returns:
        List of comment dictionaries with author, body, and creation date
    """
    url = f"{JIRA_BASE_URL}/rest/api/3/issue/{issue_key}/comment"
    params = {"maxResults": min(max_results, 50), "orderBy": "-created"}
    response = requests.get(url, auth=JIRA_AUTH, params=params, timeout=30)
    response.raise_for_status()
    return [
        {
            "author": c.get("author", {}).get("displayName"),
            "body": _extract_text(c.get("body")),
            "created": c.get("created"),
        }
        for c in response.json().get("comments", [])
    ]


def _extract_text(adf_content: dict | None) -> str:
    """Extract plain text from Atlassian Document Format (ADF) content."""
    if not adf_content:
        return ""
    if isinstance(adf_content, str):
        return adf_content
    texts = []
    for node in adf_content.get("content", []):
        if node.get("type") == "text":
            texts.append(node.get("text", ""))
        elif "content" in node:
            texts.append(_extract_text(node))
    return " ".join(texts)
```

#### Registration

```python
# In agent/server.py
from .tools.jira_context import jira_get_issue, jira_get_comments

return create_deep_agent(
    ...
    tools=[
        ...existing tools...,
        jira_get_issue,
        jira_get_comments,
    ],
)
```

> **Tip:** If Jira is also your trigger source, see the [Software Engineering Agent Blueprint](software-engineering-agent-blueprint.md) for adding a Jira webhook endpoint alongside these context tools.

---

### Confluence

Give the agent access to internal documentation stored in Confluence.

#### Prerequisites

- Confluence Cloud or Server instance
- API token with read access
- `requests` (already available)

#### Environment variables

```bash
CONFLUENCE_BASE_URL="https://your-org.atlassian.net/wiki"
CONFLUENCE_USER_EMAIL="service-account@your-org.com"
CONFLUENCE_API_TOKEN="your-api-token"
```

#### Tool implementation

```python
# agent/tools/confluence_context.py
import os
import re
import requests
from typing import Any

CONFLUENCE_BASE_URL = os.environ.get("CONFLUENCE_BASE_URL", "")
CONFLUENCE_AUTH = (
    os.environ.get("CONFLUENCE_USER_EMAIL", ""),
    os.environ.get("CONFLUENCE_API_TOKEN", ""),
)


def confluence_search(query: str, space_key: str = "", max_results: int = 5) -> list[dict[str, Any]]:
    """Search Confluence pages by title or content.

    Use this to find internal documentation, architecture decisions,
    runbooks, or design docs relevant to the current task.

    Args:
        query: Search query (CQL text search)
        space_key: Optional Confluence space key to limit search (e.g. "ENG", "PLATFORM")
        max_results: Maximum number of results to return (default: 5, max: 20)

    Returns:
        List of matching pages with title, space, URL, and excerpt
    """
    cql = f'type=page AND text ~ "{query}"'
    if space_key:
        cql += f' AND space="{space_key}"'

    url = f"{CONFLUENCE_BASE_URL}/rest/api/content/search"
    params = {"cql": cql, "limit": min(max_results, 20), "expand": "space,metadata.labels"}
    response = requests.get(url, auth=CONFLUENCE_AUTH, params=params, timeout=30)
    response.raise_for_status()
    return [
        {
            "title": r["title"],
            "space": r.get("space", {}).get("name"),
            "url": f"{CONFLUENCE_BASE_URL}{r.get('_links', {}).get('webui', '')}",
            "id": r["id"],
        }
        for r in response.json().get("results", [])
    ]


def confluence_get_page(page_id: str) -> dict[str, Any]:
    """Get the content of a Confluence page by its ID.

    Returns the page title and body as plain text. Use this after
    confluence_search to read the full content of a relevant page.

    Args:
        page_id: The Confluence page ID (numeric string)

    Returns:
        Dictionary with page title, space, and body text
    """
    url = f"{CONFLUENCE_BASE_URL}/rest/api/content/{page_id}"
    params = {"expand": "body.storage,space"}
    response = requests.get(url, auth=CONFLUENCE_AUTH, params=params, timeout=30)
    response.raise_for_status()
    data = response.json()
    html_body = data.get("body", {}).get("storage", {}).get("value", "")
    # Strip HTML tags for a readable text representation
    plain_text = re.sub(r"<[^>]+>", " ", html_body)
    plain_text = re.sub(r"\s+", " ", plain_text).strip()
    return {
        "title": data.get("title"),
        "space": data.get("space", {}).get("name"),
        "body": plain_text[:10000],  # Limit to 10k chars to avoid token bloat
        "url": f"{CONFLUENCE_BASE_URL}{data.get('_links', {}).get('webui', '')}",
    }
```

#### Registration

```python
from .tools.confluence_context import confluence_search, confluence_get_page

tools=[
    ...existing tools...,
    confluence_search,
    confluence_get_page,
]
```

---

### Postgres

Let the agent run read-only queries against a Postgres database for diagnostic context — schema inspection, recent errors, configuration values, etc.

#### Prerequisites

- A Postgres database accessible from the Open SWE process or sandbox
- A **read-only** database user with access limited to the schemas/tables the agent needs
- `psycopg2-binary` or `asyncpg` installed (`pip install psycopg2-binary`)

#### Environment variables

```bash
POSTGRES_CONTEXT_DSN="postgresql://readonly_user:password@host:5432/mydb"
POSTGRES_ALLOWED_SCHEMAS="public,analytics"   # Comma-separated allowlist
POSTGRES_MAX_ROWS=100                          # Max rows per query
```

#### Tool implementation

```python
# agent/tools/postgres_context.py
import os
import re

import psycopg2
import psycopg2.extras

POSTGRES_DSN = os.environ.get("POSTGRES_CONTEXT_DSN", "")
ALLOWED_SCHEMAS = set(
    s.strip() for s in os.environ.get("POSTGRES_ALLOWED_SCHEMAS", "public").split(",")
)
MAX_ROWS = int(os.environ.get("POSTGRES_MAX_ROWS", "100"))


def postgres_query(query: str, params: list | None = None) -> dict:
    """Run a read-only SQL query against the project database.

    Use this to look up configuration values, inspect schema metadata,
    check recent error logs, or retrieve diagnostic data.

    IMPORTANT: Only SELECT queries are allowed. The database user has
    read-only access. Queries are limited to allowed schemas and capped
    at a maximum number of rows.

    Args:
        query: SQL SELECT query. Must start with SELECT or WITH.
            Use $1, $2, etc. for parameterized values.
        params: Optional list of parameter values for the query

    Returns:
        Dictionary with column names and rows
    """
    # Safety checks
    normalized = query.strip().upper()
    if not (normalized.startswith("SELECT") or normalized.startswith("WITH")):
        return {"error": "Only SELECT and WITH (CTE) queries are allowed"}

    # Block dangerous patterns
    dangerous = re.compile(
        r"\b(INSERT|UPDATE|DELETE|DROP|ALTER|CREATE|TRUNCATE|GRANT|REVOKE)\b",
        re.IGNORECASE,
    )
    if dangerous.search(query):
        return {"error": "Query contains disallowed SQL keywords"}

    try:
        with psycopg2.connect(POSTGRES_DSN) as conn:
            conn.set_session(readonly=True, autocommit=True)
            with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
                # Validate schema names against allowlist before setting search_path
                safe_schemas = [s for s in ALLOWED_SCHEMAS if re.fullmatch(r"[a-zA-Z_][a-zA-Z0-9_]*", s)]
                cur.execute(f"SET search_path TO {','.join(safe_schemas)}")
                cur.execute(query, params or [])
                columns = [desc[0] for desc in cur.description] if cur.description else []
                rows = cur.fetchmany(MAX_ROWS)
                return {
                    "columns": columns,
                    "rows": [dict(r) for r in rows],
                    "row_count": len(rows),
                    "truncated": cur.rowcount > MAX_ROWS if cur.rowcount >= 0 else False,
                }
    except Exception as e:
        return {"error": str(e)}


def postgres_list_tables(schema: str = "public") -> dict:
    """List tables in a database schema.

    Use this to discover what tables are available before writing queries.

    Args:
        schema: Schema name (default: "public"). Must be in the allowed schemas list.

    Returns:
        Dictionary with table names and their column counts
    """
    if schema not in ALLOWED_SCHEMAS:
        return {"error": f"Schema '{schema}' is not in the allowed list: {ALLOWED_SCHEMAS}"}

    return postgres_query(
        """
        SELECT table_name, 
               (SELECT count(*) FROM information_schema.columns c 
                WHERE c.table_schema = t.table_schema AND c.table_name = t.table_name) as column_count
        FROM information_schema.tables t
        WHERE t.table_schema = $1 AND t.table_type = 'BASE TABLE'
        ORDER BY table_name
        """,
        [schema],
    )
```

#### Registration

```python
from .tools.postgres_context import postgres_query, postgres_list_tables

tools=[
    ...existing tools...,
    postgres_query,
    postgres_list_tables,
]
```

> **Security note:** Always use a dedicated read-only database user. Set `POSTGRES_ALLOWED_SCHEMAS` to limit which schemas the agent can access. The tool blocks non-SELECT queries at the application level and uses `readonly=True` at the connection level.

---

### Snowflake

Query a Snowflake data warehouse for analytics context, data lineage, or diagnostic information.

#### Prerequisites

- Snowflake account with a read-only user/role
- `snowflake-connector-python` installed (`pip install snowflake-connector-python`)

#### Environment variables

```bash
SNOWFLAKE_ACCOUNT="your-account.us-east-1"
SNOWFLAKE_USER="readonly_agent"
SNOWFLAKE_PASSWORD="your-password"
SNOWFLAKE_WAREHOUSE="AGENT_WH"
SNOWFLAKE_DATABASE="ANALYTICS"
SNOWFLAKE_SCHEMA="PUBLIC"
SNOWFLAKE_ROLE="AGENT_READONLY"
SNOWFLAKE_MAX_ROWS=100
```

#### Tool implementation

```python
# agent/tools/snowflake_context.py
import os
import re

import snowflake.connector

SNOWFLAKE_CONFIG = {
    "account": os.environ.get("SNOWFLAKE_ACCOUNT", ""),
    "user": os.environ.get("SNOWFLAKE_USER", ""),
    "password": os.environ.get("SNOWFLAKE_PASSWORD", ""),
    "warehouse": os.environ.get("SNOWFLAKE_WAREHOUSE", ""),
    "database": os.environ.get("SNOWFLAKE_DATABASE", ""),
    "schema": os.environ.get("SNOWFLAKE_SCHEMA", "PUBLIC"),
    "role": os.environ.get("SNOWFLAKE_ROLE", ""),
}
MAX_ROWS = int(os.environ.get("SNOWFLAKE_MAX_ROWS", "100"))


def snowflake_query(query: str) -> dict:
    """Run a read-only SQL query against the Snowflake data warehouse.

    Use this to look up data definitions, check data quality, inspect
    table schemas, or retrieve sample data for debugging.

    IMPORTANT: Only SELECT, SHOW, and DESCRIBE queries are allowed.
    The Snowflake role has read-only permissions.

    Args:
        query: SQL query. Must start with SELECT, WITH, SHOW, or DESCRIBE.

    Returns:
        Dictionary with column names, rows, and row count
    """
    normalized = query.strip().upper()
    allowed_prefixes = ("SELECT", "WITH", "SHOW", "DESCRIBE", "DESC")
    if not any(normalized.startswith(p) for p in allowed_prefixes):
        return {"error": "Only SELECT, WITH, SHOW, and DESCRIBE queries are allowed"}

    dangerous = re.compile(
        r"\b(INSERT|UPDATE|DELETE|DROP|ALTER|CREATE|TRUNCATE|GRANT|REVOKE|MERGE|COPY)\b",
        re.IGNORECASE,
    )
    if dangerous.search(query):
        return {"error": "Query contains disallowed SQL keywords"}

    try:
        with snowflake.connector.connect(**SNOWFLAKE_CONFIG) as conn:
            with conn.cursor() as cur:
                cur.execute(query)
                columns = [desc[0] for desc in cur.description] if cur.description else []
                rows = cur.fetchmany(MAX_ROWS)
                return {
                    "columns": columns,
                    "rows": [dict(zip(columns, row)) for row in rows],
                    "row_count": len(rows),
                    "truncated": len(rows) >= MAX_ROWS,
                }
    except Exception as e:
        return {"error": str(e)}


def snowflake_list_tables(schema: str = "", database: str = "") -> dict:
    """List tables in a Snowflake schema.

    Use this to discover available tables before writing queries.

    Args:
        schema: Schema name (default: configured SNOWFLAKE_SCHEMA)
        database: Database name (default: configured SNOWFLAKE_DATABASE)

    Returns:
        Dictionary with table information
    """
    db = database or SNOWFLAKE_CONFIG["database"]
    sch = schema or SNOWFLAKE_CONFIG["schema"]
    return snowflake_query(f"SHOW TABLES IN {db}.{sch}")
```

#### Registration

```python
from .tools.snowflake_context import snowflake_query, snowflake_list_tables

tools=[
    ...existing tools...,
    snowflake_query,
    snowflake_list_tables,
]
```

> **Security note:** Create a dedicated Snowflake role (e.g. `AGENT_READONLY`) with `SELECT` grants only on the databases/schemas the agent needs. Use a small warehouse to limit cost.

---

### AWS MWAA (Airflow)

Let the agent check DAG status, task logs, and pipeline metadata from Amazon Managed Workflows for Apache Airflow (MWAA) or any self-hosted Airflow instance.

#### Prerequisites

- Airflow REST API access (MWAA exposes this via the web server, or your self-hosted Airflow has the stable REST API enabled)
- For MWAA: AWS credentials with `airflow:CreateCliToken` or `airflow:CreateWebLoginToken` permission
- `boto3` installed (for MWAA) or `requests` (for self-hosted Airflow)

#### Environment variables

```bash
# For MWAA
MWAA_ENVIRONMENT_NAME="my-airflow-env"
AWS_DEFAULT_REGION="us-east-1"

# For self-hosted Airflow
AIRFLOW_BASE_URL="https://airflow.internal.your-org.com"
AIRFLOW_API_TOKEN="your-api-token"
```

#### Tool implementation (MWAA)

```python
# agent/tools/airflow_context.py
import json
import os
import base64

import boto3
import requests

MWAA_ENV_NAME = os.environ.get("MWAA_ENVIRONMENT_NAME", "")
AWS_REGION = os.environ.get("AWS_DEFAULT_REGION", "us-east-1")

# For self-hosted Airflow (non-MWAA)
AIRFLOW_BASE_URL = os.environ.get("AIRFLOW_BASE_URL", "")
AIRFLOW_API_TOKEN = os.environ.get("AIRFLOW_API_TOKEN", "")


def _get_mwaa_session() -> tuple[str, str]:
    """Get MWAA web login token and hostname."""
    mwaa = boto3.client("mwaa", region_name=AWS_REGION)
    response = mwaa.create_web_login_token(Name=MWAA_ENV_NAME)
    hostname = f"https://{response['WebServerHostname']}"
    token = response["WebToken"]
    return hostname, token


def _airflow_api_get(path: str, params: dict | None = None) -> dict:
    """Make a GET request to the Airflow REST API."""
    if MWAA_ENV_NAME:
        hostname, token = _get_mwaa_session()
        url = f"{hostname}/api/v1{path}"
        headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    else:
        url = f"{AIRFLOW_BASE_URL}/api/v1{path}"
        headers = {"Authorization": f"Bearer {AIRFLOW_API_TOKEN}", "Content-Type": "application/json"}

    response = requests.get(url, headers=headers, params=params or {}, timeout=30)
    response.raise_for_status()
    return response.json()


def airflow_get_dag(dag_id: str) -> dict:
    """Get details about an Airflow DAG.

    Returns the DAG description, schedule, owner, tags, and whether
    it is currently paused. Use this to understand pipeline context.

    Args:
        dag_id: The Airflow DAG ID (e.g. "etl_daily_pipeline")

    Returns:
        Dictionary with DAG metadata
    """
    data = _airflow_api_get(f"/dags/{dag_id}")
    return {
        "dag_id": data.get("dag_id"),
        "description": data.get("description"),
        "schedule_interval": data.get("schedule_interval"),
        "is_paused": data.get("is_paused"),
        "owners": data.get("owners"),
        "tags": [t["name"] for t in data.get("tags", [])],
        "file_token": data.get("file_token"),
    }


def airflow_list_dag_runs(dag_id: str, limit: int = 5) -> list[dict]:
    """List recent DAG runs for an Airflow DAG.

    Returns the most recent runs with their status, start/end times,
    and execution dates. Use this to check pipeline health.

    Args:
        dag_id: The Airflow DAG ID
        limit: Number of recent runs to return (default: 5, max: 25)

    Returns:
        List of DAG run dictionaries
    """
    data = _airflow_api_get(
        f"/dags/{dag_id}/dagRuns",
        params={"limit": min(limit, 25), "order_by": "-execution_date"},
    )
    return [
        {
            "dag_run_id": r.get("dag_run_id"),
            "state": r.get("state"),
            "execution_date": r.get("execution_date"),
            "start_date": r.get("start_date"),
            "end_date": r.get("end_date"),
        }
        for r in data.get("dag_runs", [])
    ]


def airflow_get_task_logs(
    dag_id: str, dag_run_id: str, task_id: str, try_number: int = 1
) -> dict:
    """Get logs for a specific Airflow task instance.

    Use this to debug failed or slow tasks in a pipeline.

    Args:
        dag_id: The Airflow DAG ID
        dag_run_id: The DAG run ID
        task_id: The task ID within the DAG
        try_number: Attempt number (default: 1)

    Returns:
        Dictionary with task status and log content (truncated to 5000 chars)
    """
    # Get task instance status
    task_data = _airflow_api_get(
        f"/dags/{dag_id}/dagRuns/{dag_run_id}/taskInstances/{task_id}"
    )

    # Get task logs
    log_data = _airflow_api_get(
        f"/dags/{dag_id}/dagRuns/{dag_run_id}/taskInstances/{task_id}/logs/{try_number}"
    )

    log_content = log_data if isinstance(log_data, str) else json.dumps(log_data)

    return {
        "task_id": task_data.get("task_id"),
        "state": task_data.get("state"),
        "start_date": task_data.get("start_date"),
        "end_date": task_data.get("end_date"),
        "duration": task_data.get("duration"),
        "try_number": task_data.get("try_number"),
        "log": log_content[:5000],  # Truncate logs to manage token usage
    }
```

#### Registration

```python
from .tools.airflow_context import airflow_get_dag, airflow_list_dag_runs, airflow_get_task_logs

tools=[
    ...existing tools...,
    airflow_get_dag,
    airflow_list_dag_runs,
    airflow_get_task_logs,
]
```

> **Security note:** For MWAA, use an IAM role/policy that only allows `airflow:CreateWebLoginToken` and `airflow:CreateCliToken`. For self-hosted Airflow, create a read-only role with Viewer permissions in Airflow RBAC.

---

### Knowledge base / Vector DB

Give the agent access to internal documentation, past incident reports, architectural decision records, or any corpus of text via semantic search over a vector database.

This example uses a generic pattern that works with any vector DB (Pinecone, Weaviate, Qdrant, Chroma, pgvector, etc.). Adapt the client initialization and query call to your specific provider.

#### Prerequisites

- A vector database populated with embedded documents
- An embedding model accessible from the Open SWE process
- The appropriate Python client installed (e.g. `pinecone-client`, `weaviate-client`, `qdrant-client`, `chromadb`)

#### Environment variables

```bash
# Example for Pinecone — adapt for your provider
VECTOR_DB_PROVIDER="pinecone"           # or "weaviate", "qdrant", "chroma", "pgvector"
PINECONE_API_KEY="your-api-key"
PINECONE_INDEX_NAME="internal-docs"
OPENAI_API_KEY="your-openai-key"        # For embeddings (or use your preferred provider)
KB_MAX_RESULTS=5
```

#### Tool implementation

```python
# agent/tools/knowledge_base.py
import os
from typing import Any

KB_MAX_RESULTS = int(os.environ.get("KB_MAX_RESULTS", "5"))


def _get_embedding(text: str) -> list[float]:
    """Generate an embedding vector for the given text.

    Adapt this to your embedding provider (OpenAI, Cohere, HuggingFace, etc.).
    """
    import openai

    client = openai.OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
    response = client.embeddings.create(input=text, model="text-embedding-3-small")
    return response.data[0].embedding


def _query_pinecone(query_embedding: list[float], top_k: int) -> list[dict]:
    """Query Pinecone vector index."""
    from pinecone import Pinecone

    pc = Pinecone(api_key=os.environ.get("PINECONE_API_KEY"))
    index = pc.Index(os.environ.get("PINECONE_INDEX_NAME", "internal-docs"))
    results = index.query(vector=query_embedding, top_k=top_k, include_metadata=True)
    return [
        {
            "title": match.metadata.get("title", ""),
            "source": match.metadata.get("source", ""),
            "content": match.metadata.get("text", "")[:2000],
            "score": match.score,
        }
        for match in results.matches
    ]


def _query_qdrant(query_embedding: list[float], top_k: int) -> list[dict]:
    """Query Qdrant vector database."""
    from qdrant_client import QdrantClient

    client = QdrantClient(
        url=os.environ.get("QDRANT_URL", "http://localhost:6333"),
        api_key=os.environ.get("QDRANT_API_KEY"),
    )
    results = client.search(
        collection_name=os.environ.get("QDRANT_COLLECTION", "internal-docs"),
        query_vector=query_embedding,
        limit=top_k,
    )
    return [
        {
            "title": point.payload.get("title", ""),
            "source": point.payload.get("source", ""),
            "content": point.payload.get("text", "")[:2000],
            "score": point.score,
        }
        for point in results
    ]


# Registry of supported providers — add your own here
_PROVIDER_REGISTRY = {
    "pinecone": _query_pinecone,
    "qdrant": _query_qdrant,
}


def knowledge_base_search(query: str, max_results: int = 5) -> list[dict[str, Any]]:
    """Search the internal knowledge base for relevant documentation.

    Use this to find architecture decisions, runbooks, incident reports,
    API documentation, or any internal docs that might help with the
    current task. Results are ranked by semantic relevance.

    Args:
        query: Natural language search query describing what you're looking for
        max_results: Maximum number of results to return (default: 5, max: 20)

    Returns:
        List of matching documents with title, source, content snippet, and relevance score
    """
    top_k = min(max_results, 20, KB_MAX_RESULTS)
    provider = os.environ.get("VECTOR_DB_PROVIDER", "pinecone")

    query_fn = _PROVIDER_REGISTRY.get(provider)
    if not query_fn:
        return [{"error": f"Unsupported vector DB provider: {provider}. Supported: {list(_PROVIDER_REGISTRY.keys())}"}]

    try:
        embedding = _get_embedding(query)
        return query_fn(embedding, top_k)
    except Exception as e:
        return [{"error": f"Knowledge base search failed: {str(e)}"}]
```

#### Registration

```python
from .tools.knowledge_base import knowledge_base_search

tools=[
    ...existing tools...,
    knowledge_base_search,
]
```

> **Extending to other providers:** Add a new `_query_<provider>()` function and register it in `_PROVIDER_REGISTRY`. The pattern is the same for all providers: embed the query, search the index, return structured results.

---

## Conditional context based on trigger source

You can register different context tools depending on where the task was triggered from:

```python
# In agent/server.py — inside get_agent()
source = config["configurable"].get("source")

base_tools = [http_request, fetch_url, web_search, list_repos, get_branch_name, commit_and_open_pr]

if source == "jira":
    context_tools = [jira_get_issue, jira_get_comments, confluence_search, confluence_get_page]
elif source == "linear":
    context_tools = [linear_comment, linear_get_issue, linear_get_issue_comments]
else:
    context_tools = []

# Always-on context tools
shared_context_tools = [postgres_query, snowflake_query, knowledge_base_search]

tools = [*base_tools, *context_tools, *shared_context_tools]
```

This keeps the toolset focused — the agent only sees tools relevant to its trigger source.

---

## Security and data minimization

### Credential management

| Practice | Why |
|---|---|
| Store all API keys/tokens in a secret manager | Avoid hardcoding credentials in code or `.env` files in production |
| Use service accounts with minimal permissions | Limit blast radius if credentials are compromised |
| Use short-lived tokens where possible | Rotate credentials frequently |
| Never pass credentials into the sandbox | Keep secrets in the Open SWE process, not the agent's shell |

### Query safety

| Practice | Why |
|---|---|
| Enforce read-only connections | Prevent accidental writes or deletes |
| Allowlist schemas, tables, and resources | Limit what the agent can query |
| Cap result sizes (rows, characters) | Prevent excessive token usage and context overflow |
| Block dangerous SQL keywords at the application level | Defense-in-depth beyond DB-level permissions |
| Set timeouts on all external calls | Prevent the agent from hanging on slow queries |

### Output minimization

| Practice | Why |
|---|---|
| Return only fields the agent needs | Reduce token usage and context pollution |
| Truncate large text fields | Prevent single responses from consuming the entire context window |
| Never return raw credentials, PII, or secrets | Even if the DB contains them, filter them out in the tool |
| Use structured (dict) returns, not raw text | Easier for the agent to parse and reason about |

---

## Testing context tools

Before registering a context tool with the agent, test it independently:

```python
# Test a tool function directly
from agent.tools.jira_context import jira_get_issue

result = jira_get_issue("ENG-1234")
print(result)  # Verify structure and content
```

Or test through the agent in local development mode:

1. Set `SANDBOX_TYPE="local"` for quick iteration
2. Add the tool to `agent/server.py`
3. Start the server with `uv run langgraph dev --no-browser`
4. Send a test message via Slack/Linear/GitHub that would require the agent to use the tool
5. Check the LangSmith trace to see how the agent called the tool and what it returned

---

## Summary

| Context source | Tool pattern | Key safety measures |
|---|---|---|
| **GitHub** | Built-in (`list_repos`, `github_comment`, shell access) | GitHub App permissions, per-user OAuth |
| **Jira** | `jira_get_issue`, `jira_get_comments` | Read-only API token, field filtering |
| **Confluence** | `confluence_search`, `confluence_get_page` | Read-only token, content truncation |
| **Postgres** | `postgres_query`, `postgres_list_tables` | Read-only user, schema allowlist, keyword blocking |
| **Snowflake** | `snowflake_query`, `snowflake_list_tables` | Read-only role, keyword blocking, small warehouse |
| **AWS MWAA** | `airflow_get_dag`, `airflow_list_dag_runs`, `airflow_get_task_logs` | IAM scoped to MWAA read, log truncation |
| **Knowledge base** | `knowledge_base_search` | Result count limits, content truncation |
