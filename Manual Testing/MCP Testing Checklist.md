# MCP Feature Testing Checklist

Manual, local-only checklist for `voiden-mcp-client` (connecting *to* an MCP server) and `voiden-mcp-tool` (serving requests *as* an MCP server). Not part of `.github/workflows/validate-void.yml` — some of this needs a real connected agent, a browser, or interactive setup the CI action can't do. Pair with `KNOWN-ISSUES.md` for anything already confirmed broken (don't re-report those; confirm they're *still* broken, or newly fixed).

Fixtures referenced below:
- `Automated Tests/MCP/DeepWiki Tool Call.void` — MCP client against a real third-party server
- `Automated Tests/MCP/Tool Decorated Request.void` — a single, simple `/tool`
- `Manual Testing/Tool Scenario/Widget Tools.void` — five tools in five different verification states, purpose-built for this checklist

---

## Part 1 — MCP Client (`mcp-connection`)

### Connecting

- [ ] `mcp-connection` + `mcpurl` + `mcpoperation` against a real remote server (`Automated Tests/MCP/DeepWiki Tool Call.void` — `read_wiki_structure`, `read_wiki_contents`)
- [ ] JSON config paste (`{"mcpServers": {...}}`) auto-fills `mcp-connection` — paste one from `Automated Tests/Importers/` or a real client config, in the app, and confirm the URL/headers land correctly
- [ ] A `command`-based (stdio) config entry surfaces a toast instead of silently doing nothing (Phase 1 has no stdio transport)
- [ ] Connecting to `localhost` (spin up `voiden-runner mcp serve --http --port 3900` from another project and point a fresh `mcp-connection` at `http://localhost:3900/mcp`)
- [ ] Bad/unreachable URL — confirm the failure is legible, not a silent hang
- [ ] Auth on the connection itself (an `auth` block alongside `mcp-connection`, e.g. `authType: bearer`) against a server that actually requires it

### Capabilities

- [ ] `capabilityType: tool` → `call_tool` (done — `Automated Tests/MCP/DeepWiki Tool Call.void`)
- [ ] `capabilityType: resource` → `read_resource` (`resourceUri`, e.g. `file:///...` against a local MCP server, or any resource-serving remote one)
- [ ] `capabilityType: prompt` → `get_prompt`
- [ ] In the **app** (not headless): open the URL field, confirm `list_tools`/`list_resources`/`list_prompts` auto-populate a dropdown and fill in `args` from the schema

### Response handling

- [ ] A tool call whose payload has `isError: true` still comes back as a normal (200-equivalent) response — assert on the body, not just pass/fail (see the `voiden-mcp-client` skill note on this)
- [ ] `assertions-table` against an MCP response body (field paths into `content[].text` / `structuredContent`)
- [ ] A slow tool call (15s+, streams `notifications/message` progress events) — confirm current behavior against `KNOWN-ISSUES.md` #5 (was failing after ~17s under beta); this is the one to re-check after any MCP-client fix ships
- [ ] Query params / path params on an `mcp-connection` section (rare, but documented — a server expecting an API version or tenant id on the URL)

### Multi-connection files

- [ ] Two sections, each its own `mcp-connection` (singleton is per-section, not per-file) — confirm both run independently under **Run All**

---

## Part 2 — MCP Tool (`voiden-mcp-tool`) — serving requests as agent-callable tools

Run these against `Manual Testing/Tool Scenario/Widget Tools.void`, which was purpose-built so every state below is already reproduced live — the checklist is really "does the tool confirm what the file already demonstrated."

### Discovery & verification (CLI, no agent needed)

- [ ] `voiden-runner tool list "Manual Testing/Tool Scenario/Widget Tools.void"` — lists all 5 tools with param/verify counts
- [ ] `voiden-runner tool verify "Manual Testing/Tool Scenario/Widget Tools.void" --profile` — confirm exactly:
  - [ ] `get_widget` → **verified**
  - [ ] `delete_widget` → **failing** (`contract-failure`)
  - [ ] `archive_widget` → **failing** (`contract-failure`)
  - [ ] `ping_widgets_api` → **unverified** ("No verification requests attached")
  - [ ] `admin_purge_widgets` → **failing**, tagged `auth-failure` — and confirm its `happy-path` row is reported as *skipped*, not evaluated
- [ ] `voiden-runner mcp serve . --check --profile` (path must be the **project root**, not the specific file — passing a single file breaks env resolution, a real gap worth knowing about) — confirm the serve decision per tool:
  - [ ] `get_widget` → `SERVED`
  - [ ] `delete_widget` → `WITHDRAWN` (onFailure: withdraw)
  - [ ] `archive_widget` → `SERVED (failing)` — description should carry a `⚠ DEGRADED` prefix (onFailure: advertise-degraded)
  - [ ] `ping_widgets_api` → `SERVED (unverified)`
  - [ ] `admin_purge_widgets` → `WITHDRAWN`
- [ ] `--cadence <tag>` filters which `toolverifies` rows actually run (all rows here are `cadence: commit` — add a `nightly`-tagged row to any tool and confirm `--cadence commit` skips it)
- [ ] `mode: sandbox` / `mode: none` on a row — confirm `none` skips that row's own execution without affecting sibling rows
- [ ] `tool verify --write` upserts a status back into the file, and a plain run afterward doesn't just trust that stale status
- [ ] `tool verify --json` — machine-readable output, same states as above

### Load-time validation (structural — should be excluded, not "failing")

Deliberately break a copy of `Manual Testing/Tool Scenario/Widget Tools.void` one way at a time and confirm each is **excluded**, not counted as failing:
- [ ] An agent-sourced param's `binds` doesn't match any `{{token}}` in the request
- [ ] A `{{token}}` in the request with no matching `toolparams` row (and not a real env var either)
- [ ] A `toolverifies` row's `sectionLabel` pointing at a section that doesn't exist
- [ ] Two tools sharing the same `name` — confirm **both** are excluded, not just the second
- [ ] `readOnlyHint: true` on a tool whose request is actually `POST`/`PUT`/`PATCH`/`DELETE`

### Actually connecting a real agent

- [ ] `voiden-runner mcp install` (or `--claude` / for Codex) registers the server and installs the run/verify/write-back skill — confirm in a fresh Claude Code session
- [ ] Connect and list tools — confirm `delete_widget` and `admin_purge_widgets` are **genuinely invisible** (not just marked withdrawn in `--check` output) — the agent should have no way to even attempt calling them
- [ ] Confirm `archive_widget` appears with its degraded warning visible to the agent before it decides whether to call it
- [ ] Confirm `ping_widgets_api` appears with its `[unverified — ...]` prefix
- [ ] Actually call `get_widget` from the agent — confirm the `id` argument prompt/schema matches `toolparams`, and `httpbin_url` is **never** something the agent can see or set
- [ ] Inspect `_meta['md.voiden/verification']` on a served tool (via an MCP inspector, or the agent's own tool-list debug output) — `{state, last_verified, commit}`
- [ ] Both serving paths: `@voiden/mcp-server` (stdio, npm-published) **and** `voiden-runner mcp serve` (stdio default, `--http --port <n>` for streamable-HTTP) — confirm identical serve/withdraw decisions from both, since they're documented as sharing one registration path
- [ ] `--http` mode: hit the server directly with a raw Streamable-HTTP client (or `curl` an `initialize` + `tools/list` call, same shape as `KNOWN-ISSUES.md` #5's repro) instead of through an agent, to see the raw served tool list

### A production-shaped variant worth trying once the above all pass

- [ ] Take `widget-tools.void` and change every `onFailure` to `advertise-degraded`, or add a `nightly`-cadence `error-contract` row to `get_widget` (a bad-input case) — confirms the checklist generalizes past this file's specific fixtures, not just this exact shape.
