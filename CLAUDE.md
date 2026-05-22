# n8n Etsy Agentic Workflow — Claude Instructions

## Project Purpose

This workspace is for building n8n workflows using Claude as a collaborator. You will receive workflow requests and use the n8n MCP server and n8n skills to design, create, and validate high-quality, production-ready n8n workflows.

**n8n environment:** Self-hosted via Docker
**Primary workflow domains:** Etsy integration, notifications/alerts, general automation
**Top priorities:** Modularity, Security, Error handling

---

## Available Tools

### n8n MCP Server

Connects directly to the running n8n instance. Requires `N8N_API_URL` and `N8N_API_KEY` for workflow management operations.

**Node Discovery & Documentation**
| Tool | Purpose |
|---|---|
| `get_node_essentials` | Returns 10–20 critical properties for a node (95% size reduction). Start here. |
| `search_node_properties` | Search within a node's config by keyword — use before loading full schemas. |
| `get_node_for_task` | Recommends the right node(s) for a described task. |
| `tools_documentation` | Browse tool categories or search by keyword for usage patterns and pitfalls. |

**Workflow Management** (requires API key)
| Tool | Purpose |
|---|---|
| `n8n_list_workflows` | List all workflows — always run this first to find reusable sub-workflows. |
| `n8n_create_workflow` | Create a new workflow from scratch. |
| `n8n_update_partial_workflow` | **Preferred update method.** Modify specific parts of a workflow (99% success rate). |
| `n8n_update_full_workflow` | Replace an entire workflow definition. Use only when restructuring completely. |
| `n8n_validate_workflow` | Check for disconnected nodes, invalid connections, missing fields. |
| `n8n_autofix_workflow` | Automatically repair common workflow issues. |
| `n8n_test_workflow` | Execute a workflow for testing. |
| `n8n_executions` | View execution history for debugging. |

**Granular Node Operations**
`addNode`, `updateNode`, `removeNode`, `moveNode`, `enableNode`, `disableNode`, `addConnection`, `removeConnection`, `addTag`, `removeTag`, `transferWorkflow`

**Critical nodeType format difference:**
- Use `"nodes-base.slack"` when calling search/validation tools
- Use `"n8n-nodes-base.slack"` when building/updating workflow JSON

### n8n Skills

Seven skills that activate automatically based on query context. They provide guidance, patterns, and code templates — they do not execute workflows or access n8n directly. Skills work together sequentially; a complex workflow request may activate all of them.

| Skill | Activates when... | Key outputs |
|---|---|---|
| **Expression Syntax** | Writing `{{}}` expressions | Correct syntax, variable references, webhook `.body` patterns |
| **MCP Tools Expert** | Using MCP tools to build workflows | Tool selection, command sequences, parameter formats |
| **Workflow Patterns** | Designing a new workflow | One of 5 patterns: Webhook Processing, HTTP API, Database Ops, AI Agent, Scheduled Tasks |
| **Validation Expert** | Fixing validation errors | Error severity, fix strategies, false positive handling |
| **Node Configuration** | Configuring a specific node | Operation-aware field requirements, conditional properties |
| **Code JavaScript** | Writing Code node logic | 5 production patterns, return format, best practices |
| **Code Python** | Writing Python in Code nodes | Standard library only — no `requests`, `pandas`, `numpy` |

---

## Workflow Building Process

Follow this process for every request:

1. **Clarify before building.** If the request is ambiguous, ask about the trigger, data sources, outputs, and edge cases before using any tools.
2. **Audit existing workflows.** Run `n8n_list_workflows` to find reusable sub-workflows before building new ones.
3. **Look up nodes correctly.** Use `get_node_essentials` first, then `search_node_properties` for specifics. Only fetch full schemas if needed.
4. **Design modularly.** Break complex flows into focused sub-workflows. Each workflow = one responsibility.
5. **Build iteratively.** Use `n8n_update_partial_workflow` in steps rather than creating everything in one shot. Validate after each significant change.
6. **Never touch production directly.** Duplicate the workflow, develop on the copy, validate, then promote. Never use AI to edit production workflows in place.
7. **Validate and test.** Run `n8n_validate_workflow` after every build. Check for unconnected nodes, missing error branches, and invalid references. Test with `n8n_test_workflow` or a manual trigger.

---

## Enforced Best Practices

### Modularity

- Keep each workflow focused on **one responsibility**. If it does two things, split it.
- Extract reusable logic into **sub-workflows** and call them with the Execute Workflow node (e.g., `Util/Send Slack Alert`, `Util/Fetch Etsy Orders`).
- Use consistent **naming prefixes**:
  - `Etsy/` — Etsy API integrations
  - `Util/` — Reusable utility sub-workflows
  - `Alert/` — Notification and alerting workflows
  - `Sync/` — Data sync and ETL workflows

### Security

- **Never hardcode credentials, tokens, or API keys** in workflow nodes or expressions. Always use the n8n credential store.
- Use **environment variables** (configured in Docker `.env`) for instance-level config such as base URLs, shop IDs, and feature flags. Reference them with `$env.VARIABLE_NAME`.
- Apply **least-privilege** — only request the OAuth scopes or API permissions the workflow actually needs.
- **Do not log sensitive data** in sticky notes, error messages, or Set node outputs. Mask or omit PII and secrets.
- Session data from the MCP server contains plaintext API keys — never persist MCP session data to disk unencrypted.

### Error Handling

- Every workflow **must** have an Error Trigger node or a dedicated error output branch — never optional.
- Set **retry on failure** with exponential backoff on all nodes calling external APIs (Etsy, Slack, etc.).
- Route all failures to `Alert/Workflow Failure` which must send a notification (Slack or email) containing:
  - Workflow name
  - Failing node name
  - Error message
  - Timestamp
- Never let failures pass silently.

### Etsy-Specific Rules

- **Rate limits:** Use **Wait nodes** between calls when iterating over batches. Default: 1-second wait between iterations.
- **OAuth 2.0:** Store all tokens in the n8n credential store. Never manage token refresh manually inside a workflow.
- **Pagination:** Always loop using offset/limit until all pages are exhausted. Never assume a single API call returns all results.
- **Idempotency:** Use order IDs or listing IDs to deduplicate. Workflows that process orders or listings must be safe to re-run.

---

## n8n Expression Syntax Rules

- Always wrap expressions in `{{}}`.
- Webhook payload data is **not** at root — access it via `{{$json.body.fieldName}}`, not `{{$json.fieldName}}`.
- Use bracket notation for fields with spaces: `{{$json['field name']}}`.
- In Code nodes, use JavaScript directly — do **not** use `{{}}` syntax inside Code nodes.
- For Code node return format, always return an array of objects with a `json` property: `return items.map(item => ({ json: item }))`.

---

## Workflow Structure Conventions

- The **trigger node** is always first and named descriptively: e.g., `Webhook: New Etsy Order`, `Schedule: Daily Inventory Sync`.
- **Node names** must be human-readable. Never leave defaults like `HTTP Request1` or `Set2`.
- Add **sticky notes** to explain non-obvious logic, business rules, or API quirks.
- **Tag** every workflow: `etsy`, `production`, `utility`, `alert`, `sync` — as applicable.

---

## Post-Build Verification Checklist

Run after every create or update:

- [ ] Workflow created/updated via MCP confirmed
- [ ] `n8n_validate_workflow` passes with no errors
- [ ] All nodes connected — no orphaned nodes
- [ ] Error Trigger or error branch wired up
- [ ] All credentials use the n8n credential store (no hardcoded values)
- [ ] Node names are descriptive
- [ ] Workflow tagged appropriately
- [ ] Tested with sample payload or manual trigger
- [ ] Retry settings on all external API nodes
