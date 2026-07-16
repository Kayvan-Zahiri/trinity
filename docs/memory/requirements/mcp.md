# Requirements — MCP Server & Agent Interop (A2A, Per-Agent Exposure)

> Part of Trinity's requirements set. Index & write-path rule: [requirements.md](../requirements.md).

---

## 7. MCP Server

### 7.1 Trinity MCP Server
- **Status**: ✅ Implemented
- **Description**: Agent orchestration via Model Context Protocol
- **Key Features**: FastMCP with Streamable HTTP, 62 tools, API key authentication
- **Flow**: `docs/memory/feature-flows/mcp-orchestration.md`

### 7.2 Per-User API Keys
- **Status**: ✅ Implemented
- **Description**: Generate, revoke, and track usage per key

### 7.3 MCP Execution Query Tools (MCP-007)
- **Status**: ✅ Implemented (2026-03-25)
- **Requirement ID**: MCP-007
- **GitHub Issue**: #19
- **Description**: MCP tools for querying execution history, polling async results, and monitoring agent activity
- **Key Features**: `list_recent_executions`, `get_execution_result`, `get_agent_activity_summary`; enables async polling pattern for agent-to-agent collaboration beyond 60s MCP timeout
- **Spec**: `docs/requirements/MCP_EXECUTION_QUERY_TOOLS.md`

### 7.4 Configurable MCP Server URL (MCP-URL-001)
- **Status**: ✅ Implemented (2026-03-25)
- **Requirement ID**: MCP-URL-001
- **GitHub Issue**: #76
- **Description**: Admin-configurable MCP server URL displayed on the API Keys page connection snippets. Replaces hardcoded `http://{hostname}:8080/mcp` which is wrong for production deployments where MCP is proxied through nginx.
- **Key Features**: `GET/PUT/DELETE /api/settings/mcp-url` endpoints, URL validation (requires `http(s)://` and `/mcp` suffix), Settings UI section with save/reset, auto-detect fallback when not configured
- **Flow**: `docs/memory/feature-flows/platform-settings.md`

### 7.5 Per-Agent MCP Connector (ent#46; OSS-core since #118)
- **Status**: ✅ Implemented — OSS-core (relocated from the enterprise submodule by #118; originally ent#46/#55/#51)
- **GitHub Issue**: trinity-enterprise#118 (OSS-core move); ent#46 / ent#55 (original)
- **Description**: Expose a single agent as a per-agent MCP connector — an end user adds it to their AI client (Claude Code, Cursor, Claude Desktop) in one line, turning the agent's `user_invocable` playbooks into MCP tools. The agent holds all credentials server-side; only a scoped, revocable key reaches the client. Available in **every edition** (no `mcp_connector` entitlement).
- **Key Features**:
  - Owner CRUD under `/api/agents/{name}/connector*`: `GET`/`PUT` config (enable toggle + exposed-playbook allow-list), `POST /connector/key` (mint/regenerate — secret returned once, auto-enables), `DELETE /connector/key` (revoke). `GET /connector/playbooks` is connector-key-readable.
  - Scoped key = a row in the OSS `mcp_api_keys` table with `scope='connector'`, bound to the agent; validated by the existing OSS auth fence (`dependencies._enforce_connector_scope`) which fences a connector key to exactly `POST /{agent}/chat` + `GET /{agent}/connector/playbooks`.
  - Per-client copy-paste setup snippets (`services/connector_service.build_snippets`): Claude Code CLI + `.mcp.json`, Cursor, Claude Desktop.
  - Exposed-playbook allow-list (ent#55): `enterprise_connectors.exposed_playbooks` JSON array (NULL ⇒ all `user_invocable`); `user_invocable:false` playbooks are **never** exposed even if listed; `automation:gated` passed through as advisory metadata.
  - MCP proxy tools (`src/mcp-server/src/tools/connector.ts`, already OSS): `list_playbooks`, `run_playbook`, `ask` — visible only to `scope='connector'` sessions.
  - UI: `ConnectorChannelPanel.vue` + `ExposedToolsPanel.vue` in the Sharing tab, shown to all agent owners (un-gated).
- **Schema**: `enterprise_connectors` (name kept for zero-migration adoption of existing enterprise installs) — dual-track (SQLite `db/migrations.py:enterprise_connectors_table` + Alembic `0015_enterprise_connectors`).
- **Deferred (Part B, blocked on #848 design sign-off)**: email-auth onboarding — inline `request_login`/`verify_login` MCP tools so an external user on the agent's sharing allow-list connects with just their email (no pre-minted key). Tracked separately.
- **Flow**: `docs/memory/feature-flows/mcp-connector.md`

---

## 32. A2A Agent Discoverability (#737)

### 32.1 A2A Agent Card Endpoint (#737 — Phase 1)
- **Status**: ✅ Implemented (Phase 1)
- **Implements**: Issue #737
- **Description**: Each Trinity agent exposes an A2A-protocol Agent
  Card so external orchestrators (AWS Bedrock, Azure Copilot, Google
  ADK) can discover its identity, skills, and auth requirements
  without knowing Trinity's internal API. A2A is Google's open
  agent-interoperability protocol (https://google.github.io/A2A/).
- **Endpoint**: `GET /api/agents/{name}/a2a/agent-card` — returns a
  card built from the agent's `template.yaml` (`name`, `description`,
  `version`, `skills[]` mapped from `capabilities[]` with
  `use_cases[]` as examples) plus declared
  `securitySchemes.bearerAuth` (Trinity MCP API key) and
  `capabilities.streaming = true`. Auth-gated by `AuthorizedAgentByName`.
- **Protocol version**: `protocolVersion: "0.3.0"` (ent#157). The
  earlier `"1.0"` was a placeholder with no endpoint behind it; the
  card now points at a JSON-RPC server that speaks the v0.3.x method
  set, so the advertised version has to match what answers.
- **Behavior**: card data fetched from the agent-server's
  `/api/template/info`; falls back to Docker labels when the agent
  is stopped or unreachable (never 5xx's). The `url` field points at
  `POST /a2a/{name}` — the JSON-RPC task endpoint (§32.2) — replacing
  the public-chat placeholder it carried before that endpoint existed.
- **Phase 2 (deferred)**: Redis caching of the card; auth-gated
  extended card with internal endpoint URLs + full skill schemas; MCP
  `get_agent_card` tool. (The host-root well-known route and the
  JSON-RPC server this card addresses shipped in §32.2.)

### 32.2 A2A Inbound Server — public card + JSON-RPC/SSE (ent#157)
- **Status**: 🚧 In Progress
- **Implements**: trinity-enterprise#157
- **Description**: The serving half of A2A — §32.1 describes an agent,
  this lets an external orchestrator *task* it. A spec-shaped public
  discovery route plus a JSON-RPC 2.0 task endpoint, so an A2A client
  discovers a Trinity agent and drives it with no Trinity-specific code.
- **FR-1 — Exposure is opt-in, default OFF**: `agent_ownership.a2a_exposed
  INTEGER DEFAULT 0`. Nothing is publicly reachable until an owner
  turns it on. Dual-track migration (SQLite
  `agent_ownership_a2a_exposed` + Alembic `0024`).
- **FR-2 — Public discovery route**: `GET /a2a/{name}/.well-known/agent-card.json`
  (unauthenticated, prefix-less `a2a_server_router`). Non-exposed,
  non-existent, and inaccessible agents return a **uniform 404** so the
  public surface is not an enumeration oracle (Invariant #8).
- **FR-3 — JSON-RPC task endpoint**: `POST /a2a/{name}` — `message/send`,
  `message/stream` (SSE), `tasks/get`, `tasks/cancel`;
  `tasks/resubscribe` returns an explicit unsupported error. Bearer auth
  is a Trinity MCP API key, fail-closed 401 before any envelope parse.
  Protocol errors ride the JSON-RPC envelope at HTTP 200; auth is the
  deliberate exception.
- **FR-4 — Per-caller dedup (Invariant #18)**: a repeated `messageId`
  must not double-execute, but `messageId` is **peer-controlled**,
  auto-generated by SDKs, and the spec only requires uniqueness *per
  client* — so the dedup scope is keyed on `(agent, caller principal)`,
  never the agent alone. An agent-only scope lets one caller's
  `messageId` resolve to another caller's stored snapshot, disclosing
  the agent's response text and silently skipping the second caller's
  task. The principal prefers the MCP key id over the username, because
  agent-scoped keys all resolve to a single owner user.
- **FR-5 — Public route is rate limited**: the discovery route is
  unauthenticated and its URL is published by design, while each hit
  costs a DB read, a live Docker API call, and an HTTP call into the
  container. Per-IP limit (60/min) enforced **before** that work, via
  the shared `services/rate_limiter` and the trusted-proxy-aware client
  IP. The JSON-RPC body is capped before parsing.
- **FR-6 — Cancellation is honest**: `tasks/cancel` reports what
  actually happened — a queued row cancels through the backlog CAS, a
  terminal row returns `TaskNotCancelable`, and a failed terminate is
  surfaced rather than reported as success. Telling a caller "canceled"
  for a task that later drains, runs, and bills is a correctness bug,
  not a cosmetic one (cf. #1082 status-as-projection).
- **FR-7 — Allow-list seam (open-core)**: `services/a2a_gate.py` lets a
  registered provider further restrict *which* caller identities may
  task an exposed agent. OSS registers none → any authenticated
  owner/shared caller is allowed. The gate **fails open** — a provider
  error never blocks an authenticated caller — so it is a restriction
  layered on authentication, not a security boundary. The identity
  compared is the caller's account email (username fallback).
- **Front door**: nginx proxies `/a2a/` to the backend with SSE
  buffering off, so external clients reach Trinity through the same
  door as `/api/` rather than `:8000` directly.
- **Flow**: `docs/memory/feature-flows/a2a-inbound-server.md`

### 32.3 A2A Control over MCP (ent#160)
- **Status**: 🚧 In Progress
- **Implements**: trinity-enterprise#160
- **Description**: Manage A2A exposure, inspect the card, and edit the
  inbound allow-list from an MCP client — so an operator configures
  interop without the web UI. Tools live in `src/mcp-server/src/tools/a2a.ts`
  (Invariant #13: backend router + MCP tool stay in sync).
- **Surface**: read the exposure state, card URL, advertised skills,
  allow-list, and registered outbound endpoints; toggle exposure; add
  and remove inbound identities. Mutating tools are owner-gated at the
  backend; the MCP layer applies the same `{self} ∪ permitted` gate for
  agent-scoped keys as the rest of the tool surface.
- **UI counterpart**: `components/A2aPanel.vue` on the agent's Sharing
  tab.

---

## 45. Per-Agent MCP Exposure — Dedicated Dynamic Tools (#846)

**Description**: A per-agent owner-toggled flag (`mcp_exposed`, default off) that publishes
an agent as a first-class MCP tool. When enabled, the Trinity MCP server dynamically
registers a dedicated `chat_with_<slug>` tool — functionally identical to `chat_with_agent`
with the agent name pre-filled — so a curated, well-known agent surfaces as a named tool
instead of requiring `list_agents` + `chat_with_agent`. Toggling adds/removes the tool at
runtime with **no MCP-server restart**. The flag publishes a surface only; execution always
runs the same access gate, so ownership/sharing is never bypassed.

- **FR-1 — Toggle**: `agent_ownership.mcp_exposed INTEGER DEFAULT 0`; owner-only `GET`/`PUT
  /api/agents/{name}/mcp-exposed`. PUT refuses the system agent (403). Getter/setter both guard
  `deleted_at IS NULL` (a soft-deleted agent can never be flipped exposed). Dual-track migration
  (SQLite `agent_ownership_mcp_exposed` + Alembic `0009`).
- **FR-2 — Canonical slug (single backend source of truth)**: the backend computes the
  deterministic, collision-free `tool_name` over the full exposed set (sorted; sanitized
  `chat_with_<slug>`; `_<sha1(name)[:4]>` suffix on agent-vs-agent base-slug collision). The
  per-agent GET and the internal poll endpoint use the same helper, so UI and MCP never
  diverge.
- **FR-3 — Internal poll endpoint**: `GET /api/internal/mcp-exposed-agents` (`X-Internal-Secret`)
  returns `{agent_name, tool_name, description}` per exposed agent. `description` is generated
  from cheap Docker `trinity.template` label metadata (no container read; works for stopped
  agents).
- **FR-4 — Refresh = poll**: the MCP server polls the internal endpoint (~20s), diffs an
  `agentName→toolName` map, and calls FastMCP `addTool`/`removeTool`; FastMCP fans
  `notifications/tools/list_changed` to live sessions. The reconciler is **fail-open** (mutate
  only on a valid 200; keep last-known set otherwise) and holds an in-flight mutex. A final
  guard skips any `tool_name` colliding with a built-in tool.
- **FR-5 — No logic fork**: the `chat_with_agent` body is extracted into a shared
  `runAgentChat`, reused by `chat_with_agent` and every dedicated tool (preserves #946 pull
  routing, parallel/self-task paths, idempotency tokens, #914 gateway-timeout recovery, access
  denial). Dedicated tools register with the `connectorDenied` visibility gate and bind their
  audit target (no `agent_name` param).
- **FR-6 — Surfacing**: `mcp_exposed` is exposed on `GET /api/agents` / MCP `list_agents`. A
  Settings-tab toggle ("Expose via MCP") shows the computed tool name and up-to-poll-interval
  latency copy.

**Deferred**: WS push (poll latency ≤20s is fine for an owner-toggled flag); a partial index on
`mcp_exposed`; tool-name stability across an agent rename (rename re-slugs); multi-replica MCP
servers (each replica polls + reconciles independently).

---

## Trinity Helper MCP Server (#1459)

**Description**: A standalone, dependency-light MCP server (`src/helper-mcp/`, npm
`@abilityai/trinity-docs-mcp`) that exposes the public Trinity Docs Q&A service
(DOCS-QA-001, `docs/memory/feature-flows/trinity-docs-qa.md`) as MCP tools, so anyone can
add a grounded "ask Trinity anything" assistant to Claude Code / Claude Desktop / any MCP
client **without running a Trinity instance**. Pure protocol adapter over the existing
`ask-trinity` Cloud Function — no new backend/QA logic, no authentication, no credentials.
Distinct from the main Trinity MCP server (`src/mcp-server/`, requires a Trinity API key).

- **FR-1 — `ask_trinity` tool**: `{question (required, ≤4,000 chars), session_id?
  (opaque string)}` → POSTs the public endpoint; returns the answer plus the response
  `session_id` for multi-turn follow-ups. Session expiry is **silent** server-side (an
  expired/invalid id yields a NEW session with HTTP 200/`SUCCEEDED`) — the tool always
  returns the effective session_id and appends a context-lost warning when it differs
  from the input.
- **FR-2 — `get_agent_requirements` tool**: fetches `docs/TRINITY_COMPATIBLE_AGENT_GUIDE.md`
  from raw.githubusercontent.com at call time (living doc, no bundling staleness); on fetch
  failure returns a static quick-reference fallback + the GitHub URL, never an error-only
  response. Same tool name/shape as the main MCP server's (per-server namespacing).
- **FR-3 — Robustness**: 50s abort timeout (under the 60s MCP client default), no
  auto-retry, `redirect: "error"`, non-JSON response guard, structured error text for
  non-200 / `state != SUCCEEDED` / empty answer — a tool call never crashes the server.
  `session_id` is an opaque string end-to-end (live values exceed 2^53; numeric handling
  would corrupt them).
- **FR-4 — Distribution**: npx-runnable stdio package; runtime deps = official
  `@modelcontextprotocol/sdk` + `zod` only (deliberately NOT fastmcp — smaller
  supply-chain surface); `console.error`-only logging (stdout is the JSON-RPC channel);
  Node ≥18 guarded at startup. Publish via `.github/workflows/publish-helper-mcp.yml`
  (npm provenance; one-time manual first publish creates the package, then trusted
  publishing takes over).
- **FR-5 — Endpoint override**: `ASK_TRINITY_ENDPOINT` env var (default: the public Cloud
  Function URL) for self-hosted mirrors and the CI smoke test; logged to stderr when set.
- **FR-6 — Corpus**: the docs-sync workflow indexes `docs/onboarding/**`,
  `docs/user-docs/**` (incl. the 264-Q&A FAQ) and `docs/TRINITY_COMPATIBLE_AGENT_GUIDE.md`
  so answers cover evaluator/operator questions, not just onboarding.

**Deferred**: hosted remote Streamable-HTTP endpoint + vanity URL + MCP registry listing
(fast-follow; the official SDK keeps the transport option open); Cloud Function citations
passthrough (the endpoint returns no citations today — the adapter forwards a `citations`
field if it ever appears); #1460 (`ask_trinity` inside the main Trinity MCP server —
shares the same tool name/schema and endpoint-client contract).
