# Software Engineering Agent Blueprint (Jira → PR → Review → Production)

This guide shows how to build and deploy an Open SWE-based agent that can:

1. Get tickets from Jira  
2. Pull context from Jira, GitHub, Confluence, internal knowledge base, Postgres, Snowflake, and AWS  
3. Implement changes and open pull requests  
4. Address human review comments  
5. Merge approved changes to production

It is designed to be used with:

- [INSTALLATION.md](../INSTALLATION.md)
- [CUSTOMIZATION.md](../CUSTOMIZATION.md)

---

## 1) Target architecture

Use Open SWE as the orchestration/runtime layer and add enterprise integrations as trigger handlers and tools.

- **Trigger/ingestion layer**: Jira webhooks (issue created/updated/commented)
- **Context layer**: tool functions for Jira/GitHub/Confluence/KB/Postgres/Snowflake/AWS
- **Execution layer**: Open SWE sandbox + deepagents workflow
- **Delivery layer**: `commit_and_open_pr` + GitHub review-comment handling
- **Release layer**: merge gate (manual approval, merge queue, or automated merge middleware)

---

## 2) End-to-end workflow

### A. Jira ticket intake

1. Add a Jira webhook endpoint in `agent/webapp.py` (same pattern as existing webhook handlers).
2. Validate webhook signatures and reject unauthorized requests.
3. Extract:
   - Jira issue key
   - summary/description
   - labels/components/priority
   - assignee/reporter identity
   - target repository (from custom field, mapping, or ticket annotation)
4. Generate a deterministic Open SWE thread id from Jira issue key.
5. Start an agent run with `configurable` metadata:
   - `source: "jira"`
   - `repo: {"owner": "...", "name": "..."}`
   - ticket metadata for prompt/tool use

### B. Multi-source context retrieval

For each source, expose a narrowly-scoped tool in `agent/tools/`:

- Jira: issue details, linked issues, comments, transitions
- GitHub: repo files/issues/PR context (existing GitHub tools + API helpers)
- Confluence: page lookup by title/label/space and content retrieval
- Knowledge base: search/read APIs for internal docs
- Postgres: parameterized read-only queries
- Snowflake: parameterized read-only queries
- AWS: read-only context tools (for example SSM Parameter Store, CloudWatch, S3 metadata)

Design recommendations:

- Keep all DB and cloud tools **read-only** by default.
- Enforce allowlists for endpoints, schemas, databases, tables, and AWS resources.
- Add query/result limits and timeouts.
- Require explicit arguments (no unconstrained raw query execution).
- Return concise, structured outputs to reduce token usage.

### C. Ticket implementation and PR creation

1. Agent clones/updates repo in sandbox.
2. Agent implements changes.
3. Agent runs repo validation (lint/tests/build according to repo conventions).
4. Agent calls `commit_and_open_pr` to push branch and open PR.
5. Agent posts back to Jira with PR URL and status.

### D. Human review comments loop

Open SWE already supports GitHub PR feedback via webhook events (`pull_request_review` and `pull_request_review_comment`).

Recommended pattern:

1. Reviewer comments on PR.
2. GitHub webhook re-invokes agent on the same thread.
3. Agent fetches unresolved/new comments.
4. Agent applies fixes and updates PR branch.
5. Agent comments with resolution summary.
6. Repeat until approval.

### E. Merge to production

Choose one merge strategy:

- **Preferred**: branch protection + required checks + GitHub merge queue/auto-merge.
- **Optional automation**: add a merge middleware/tool that merges only when:
  - all required checks pass
  - required approvals are present
  - branch protection conditions are satisfied

Production safety controls:

- Require protected branches and status checks.
- Restrict who/what can merge.
- Use environment protection rules for deploy workflows.
- Record trace/run IDs in Jira updates for auditability.

---

## 3) Implementation map in this repository

Use these files as extension points:

- `agent/webapp.py`
  - Add Jira webhook endpoint and background processing function
  - Reuse deterministic thread-id and run creation patterns already used for Linear/Slack/GitHub
- `agent/server.py`
  - Register new enterprise tools in `tools=[...]`
  - Keep middleware chain for message handling and PR safety net
- `agent/tools/`
  - Add source-specific tools (Jira/Confluence/KB/Postgres/Snowflake/AWS)
- `agent/utils/`
  - Shared clients/auth helpers, signature verification, repo routing

---

## 4) Security and governance requirements

For enterprise deployment, enforce all of the following:

1. **Authentication/authorization**
   - Store credentials in secret manager
   - Use least-privilege service accounts/roles
   - Use short-lived tokens where possible

2. **Data minimization**
   - Fetch only required fields
   - Mask/redact secrets and PII in tool responses
   - Avoid writing sensitive content to PR comments

3. **Execution controls**
   - Keep sandbox isolation enabled in production
   - Restrict outbound network destinations
   - Enforce command and runtime limits

4. **Auditability**
   - Correlate Jira issue key, run ID, PR URL, commit SHA, and deploy artifact
   - Emit structured logs for all external tool calls

---

## 5) Deployment model

1. Complete base setup from `INSTALLATION.md`.
2. Deploy on LangGraph Cloud (or equivalent managed environment).
3. Configure production environment variables for:
   - GitHub App
   - Jira credentials and webhook secret
   - Confluence/KB API tokens
   - Postgres and Snowflake read-only credentials
   - AWS role/credentials
4. Point Jira and GitHub webhooks to production webhook endpoints.
5. Enable monitoring/alerts on:
   - webhook failures
   - tool-call failures
   - stuck/long-running agent tasks
   - PR creation/merge failures

---

## 6) Suggested phased rollout

### Phase 1: Core developer loop

- Jira intake
- GitHub repo context
- PR creation
- PR review comment handling

### Phase 2: Deep context

- Confluence + internal KB tools
- Postgres/Snowflake read-only diagnostics
- AWS operational context lookups

### Phase 3: Release automation

- Merge queue integration
- Optional guarded auto-merge
- Automatic Jira status transitions (In Progress → In Review → Done)

---

## 7) Acceptance checklist

Use this checklist to validate your implementation:

- [ ] Jira tickets can trigger a run and route to the correct repository
- [ ] Agent can retrieve context from all required sources
- [ ] Agent can make code changes and open PRs
- [ ] Agent can process and resolve human PR review comments
- [ ] Approved PRs can be merged through your production-safe merge path
- [ ] All actions are auditable and tied to ticket/run/PR identifiers

