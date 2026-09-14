# Known Issues

Found by actually *running* this repo end to end, not just authoring it —
first against `@voiden/runner@2.2.0` (npm `latest`) on 2026-09-14, then
against `@voiden/runner@2.3.0-beta.19` (npm `beta`, the same version stamped
in every file's frontmatter here) the same day, per request. **This repo now
targets the beta CLI** — see the README. Every item below was reproduced
live — repro commands included — and each corresponding `assertions-table`
row is disabled (not deleted) with a pointer back here, so the suite stays
green while the gap is tracked. Re-enable a row and delete its item here once
the fix ships.

Environment: macOS (darwin), Node v22.21.0.

**Switching runner versions leaves stale plugin bundles behind — always
`voiden-runner plugin update --all` right after.** `voiden-rest-api` and
`voiden-advanced-auth` etc. are cached separately from the core npm package
(under `~/.voiden/extensions`), pinned to whatever was current when first
installed. Installing `@voiden/runner@beta` over `@voiden/runner@2.2.0` did
**not** update them automatically — the old `voiden-rest-api@1.4.12` running
under the new beta core produced what looked at first like a severe new
regression (headers/query/path-table completely non-functional headlessly).
Running `voiden-runner plugin update --all` (→ `voiden-rest-api@1.4.13`,
`voiden-scripting@1.1.8`) fixed it instantly — it was a stale-cache mismatch,
not a runner bug. Worth a UX fix upstream (warn or auto-update on a core
version change) so this isn't a trap for the next person who upgrades.

---

## 0. ⚠️ A failed `assertions-table` row never fails the run — exit code stays 0

**Still present in beta.** The single most important finding here — it's the
exact mechanism Samuel's "GitHub Actions to validate the void files" idea
depends on, and it silently doesn't work: none of `--bail`,
`--stop-on-failure`, nor `--fail-on-error` react to a failed assertion — only
a request-level failure (network error, unresolved variable, timeout) affects
the exit code or the printed/JSON `summary.passed`/`summary.failed` counts.

**Repro:**
```bash
mkdir /tmp/t && cd /tmp/t
cat > bad.void << 'VOID'
```void
type: request
attrs: { uid: "11111111-1111-1111-1111-111111111111" }
content:
  - type: method
    attrs: { uid: "22222222-2222-2222-2222-222222222222", method: GET, visible: true }
    content: GET
  - type: url
    attrs: { uid: "33333333-3333-3333-3333-333333333333" }
    content: "https://httpbin.org/status/500"
```
```void
type: assertions-table
attrs: { uid: "44444444-4444-4444-4444-444444444444" }
content:
  - type: table
    rows:
      - attrs: { disabled: false }
        row: ["should fail", "status", "equals", "200"]
```
VOID
voiden-runner run . --stop-on-failure; echo "exit: $?"
```
**Expected:** the assertion fails (500 ≠ 200) → non-zero exit, `failed: 1` in
the summary.
**Actual:** `Summary  1 request · 1 passed · 0 failed`, **`exit: 0`**. The
per-assertion result is correctly recorded in `--output-json`'s
`requests[].reportEntries[]` (`{"type":"assertion","passed":false,...}`) — the
data is right there — it's just never rolled up into `summary.failed` or the
exit code.

**Impact:** any CI pipeline built the "obvious" way — `voiden-runner run . --env .env --stop-on-failure` — goes green regardless of whether the API
actually behaved correctly. `.github/workflows/validate-void.yml` in this repo
works around it by reading `result.json` directly and failing the job on any
`reportEntries[].passed === false` — see that file's comments. That workaround
should be temporary; this belongs fixed in `@voiden/runner` itself before any
"validate void files in CI" pitch goes to teams, or every green checkmark is
misleading by default.

---

## 1. `bearer`/`basic`/`apiKey` auth send empty credentials — **beta-only regression**

**File:** `known-broken-under-beta/auth-types.void` — "Bearer Token", "Basic Auth", "API Key"
sections. **Not present in `@voiden/runner@2.2.0`** — all three worked
correctly there. Broken under `@voiden/runner@2.3.0-beta.19`, confirmed with
`voiden-advanced-auth` already at its latest registry version (no update
available via `plugin update --all`) — this looks like the beta core changed
a block-value-extraction contract that `voiden-advanced-auth` hasn't caught up
to yet, the same class of issue `voiden-rest-api@1.4.13` already fixed for
table blocks (see the note at the top of this file) — just not shipped for
this plugin yet.

**Repro:**
```
voiden-runner run known-broken-under-beta/auth-types.void --env .env --no-session --show-req
```
**Expected:** `Authorization: Bearer voiden-demo-token`,
`Authorization: Basic dm9pZGVuOnNjZW5hcmlvcw==` (voiden:scenarios), and an
`api_key=demo-api-key-456` query param, each producing a real `200` from the
matching httpbin endpoint.
**Actual:**
- Bearer → literal `Authorization: Bearer ` (empty token) → `401`.
- Basic → literal `Authorization: Basic Og==` (base64 for an empty `:`) → `401`.
- API Key → no query param appended at all.

All three row values are authored as plain literal strings in the `auth`
block's table (not `{{}}` template refs), same shape used everywhere else in
this repo, so it isn't a template-substitution issue — the table's Value
column just isn't reaching the request under this plugin/core pairing.

## 2. Digest auth doesn't complete the challenge/response round trip headlessly

**File:** `known-broken-under-beta/auth-types.void` — "Digest Auth" section. **Present in both**
`2.2.0` and `2.3.0-beta.19`.
**Repro:**
```
voiden-runner run known-broken-under-beta/auth-types.void --env .env --show-req --show-res
```
**Expected:** `voiden-runner` sends the initial request, receives httpbin's
`401` + `WWW-Authenticate: Digest ...` challenge, computes the digest, and
re-sends — ending in `200`, `body.authenticated: true`.
**Actual:** Only one request is ever sent (visible in `--show-req` — no
`Authorization` header, and no second attempt logged). The run ends on the
`401` challenge itself.

## 3. `options-table`'s `follow_redirects: false` is not respected headlessly

**File:** `rest/options-and-cookies.void` — "Redirect Handling" section.
**Present in both** `2.2.0` and `2.3.0-beta.19` (re-verified after the
`voiden-rest-api@1.4.13` update — still broken).
**Repro:**
```
voiden-runner run rest/options-and-cookies.void --env .env --show-req --show-res
```
**Expected:** `GET /redirect/1` with `follow_redirects: false` stops at
httpbin's own `302` (`Location: /get`).
**Actual:** The redirect is followed anyway — response comes back `200` from
`/get`, not `302` from `/redirect/1`.

## 4. `cookies-table` rows never reach the wire headlessly

**File:** `rest/options-and-cookies.void` — "Cookies" section. **Present in
both** `2.2.0` and `2.3.0-beta.19` (re-verified after the update — still
broken).
**Repro:** same command as #3.
**Expected:** `GET /cookies` echoes back `session_id`/`tracking` under
`body.cookies`.
**Actual:** `body.cookies` comes back `{}` — `--show-req` confirms no `Cookie`
header was ever sent.

## 5. MCP client and DeepWiki's `ask_question` tool (slow calls / SSE progress notifications)

**File:** N/A — deliberately left out of `mcp/deepwiki-tool-call.void` so the
fixture stays deterministic; found under `2.2.0`, not re-verified under beta
(the section was replaced with `read_wiki_contents` rather than re-tested,
since its backing behavior — DeepWiki's own response latency — doesn't depend
on the Voiden version). `read_wiki_structure` and `read_wiki_contents` (same
file) both work correctly and quickly (~2–5s) against the same server, on
both runner versions.
**Repro:**
```bash
curl -s -X POST https://mcp.deepwiki.com/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ask_question","arguments":{"repoName":"anthropics/claude-code","question":"What CLI commands does this project expose?"}}}'
```
This curl succeeds in ~18s — DeepWiki streams `notifications/message` progress
events ("Processing query... (15s elapsed)") before the final `result`. The
equivalent `.void` `mcp-connection` / `mcpoperation` run through
`voiden-runner run` failed after ~17s with a bare `MCP request failed` and no
response body — right around where the real answer would have arrived.
Unclear whether it's a hard client-side timeout or the interim
`notifications/message` events aren't handled — worth a closer look, since any
tool whose backing service legitimately takes 15s+ will hit the same wall.

---

## Fixed between `2.2.0` and beta — no longer issues, kept here for the record

- **`multipart-table` sending no body headlessly** — fixed by the
  `voiden-rest-api@1.4.13` update (see the stale-plugin-cache note at the top
  of this file). Confirmed working: `rest/request-bodies.void`'s "Multipart
  Form" section now correctly sends `name`/`role` and httpbin echoes them back
  in `body.form`.
- **`url-table` (form-urlencoded) sending no body headlessly** — same fix,
  same plugin update. `rest/request-bodies.void`'s "URL-Encoded Form" section
  now sends `Content-Type: application/x-www-form-urlencoded` and the correct
  `username=...&source=...` body.
- **`--profile` / `.voiden/env-*.yaml` support in the CLI** — didn't exist in
  `2.2.0` (`--profile` was `error: unknown option`, and pointing `--env`
  straight at a `.voiden/env-public.yaml` failed with `Malformed line 1 in
  .env file: missing "="`). Works correctly in beta:
  `voiden-runner run . --profile` reads `.voiden/env-public.yaml` directly, no
  flat `.env` duplication needed. This repo still ships both for now (see
  README) but the flat `.env` can likely be dropped once the team is fully on
  beta.
- **The `/tool` extension's CLI surface** — `voiden-runner tool list`,
  `tool verify`, and `mcp serve [--http|--check]` didn't exist in `2.2.0`
  (`error: unknown command 'tool'`). All present and working in beta — see
  `mcp/tool-decorated-request.void`'s header comment for the exact commands.
  One real fixture bug this surfaced: `tool verify` initially reported
  `create_customer` as `excluded` with
  `[unresolved-placeholder] Tool "create_customer" request uses
  "{{REQRES_URL}}", which isn't declared as any parameter's "binds"` — correct,
  documented load-time validation (see the `voiden-mcp` skill's "Load-time
  validation" section), not a runner bug. Fixed by adding a third
  `toolparams` row (`source: environment`) declaring `REQRES_URL` explicitly,
  same as `customer_name`/`customer_email` are declared as `source: agent`.

---

## Operational risk — not a Voiden bug, but hit repeatedly during this work

**reqres.in's anonymous tier is 40 requests/day per IP**, and this repo's own
testing hit that limit outright mid-session (`429 rate_limit_exceeded`),
blocking `crud-and-chaining/users-crud.void` and
`mcp/tool-decorated-request.void` for the rest of the day. Every fixture in
this repo that hits `reqres.in` will fail this way if the limit is already
spent by other testing on the same IP/CI runner pool — a real flakiness risk
for `validate-void.yml` specifically, since GitHub-hosted runners share IP
ranges across many unrelated workflows. Options worth considering: a
reqres.in API key (raises the limit, needs a secret), swapping the CRUD
fixture for a self-hosted mock (e.g. `json-server` spun up as a CI step, full
control but more to maintain), or just accepting occasional 429-flakiness on
that one file and treating it separately from the rest of the suite's signal.
Not fixed as part of this pass — flagging it here since it's exactly the kind
of thing that'd otherwise show up as "random CI flakiness" with no obvious
cause.
