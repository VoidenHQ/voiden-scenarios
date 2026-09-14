# Known Issues

Found while building and actually running this scenario repo end to end against
the currently-published `@voiden/runner@2.2.0` (globally installed via
`npm install -g @voiden/runner`) on 2026-09-14. Every item below was reproduced
live, not guessed at — repro commands included. This file exists so the
corresponding `assertions-table` rows could be disabled (not deleted) with a
pointer back here, keeping the suite green while the gap is tracked. Re-enable
each disabled row once its fix ships, and delete that item from this file.

Environment: macOS (darwin), Node v22.21.0, `@voiden/runner@2.2.0`.

---

## 0. ⚠️ A failed `assertions-table` row never fails the run — exit code stays 0

**This is the one to fix first.** It's the exact thing Samuel's "GitHub Actions
to validate the void files" idea depends on, and as of `@voiden/runner@2.2.0`
it silently doesn't work: none of `--bail`, `--stop-on-failure`, nor
`--fail-on-error` react to a failed assertion — only a request-level failure
(network error, unresolved variable, timeout) affects the exit code or the
printed/JSON `summary.passed`/`summary.failed` counts.

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

## 1. `multipart-table` sends no body headlessly

**File:** `rest/request-bodies.void` — "Multipart Form" section
**Repro:**
```
voiden-runner run rest/request-bodies.void --env .env --show-req
```
**Expected:** `name`/`role` fields arrive in httpbin's `body.form`.
**Actual:** The outgoing request has no `Content-Type`, `Content-Length: 0`, and
an entirely empty body — `--show-req` confirms nothing was ever attached. The
`multipart-table` block is silently dropped.

## 2. `url-table` (form-urlencoded) sends no body headlessly

**File:** `rest/request-bodies.void` — "URL-Encoded Form" section
**Repro:** same command as above.
**Expected:** `username`/`source` fields arrive in httpbin's `body.form`, with
`Content-Type: application/x-www-form-urlencoded`.
**Actual:** Same symptom as #1 — no Content-Type, no body sent at all.

## 3. `options-table`'s `follow_redirects: false` is not respected headlessly

**File:** `rest/options-and-cookies.void` — "Redirect Handling" section
**Repro:**
```
voiden-runner run rest/options-and-cookies.void --env .env --show-req --show-res
```
**Expected:** `GET /redirect/1` with `follow_redirects: false` stops at
httpbin's own `302` (`Location: /get`).
**Actual:** The redirect is followed anyway — response comes back `200` from
`/get`, not `302` from `/redirect/1`.

## 4. `cookies-table` rows never reach the wire headlessly

**File:** `rest/options-and-cookies.void` — "Cookies" section
**Repro:** same command as #3.
**Expected:** `GET /cookies` echoes back `session_id`/`tracking` under
`body.cookies`.
**Actual:** `body.cookies` comes back `{}` — `--show-req` confirms no `Cookie`
header was ever sent.

## 5. Digest auth doesn't complete the challenge/response round trip headlessly

**File:** `auth/auth-types.void` — "Digest Auth" section
**Repro:**
```
voiden-runner run auth/auth-types.void --env .env --show-req --show-res
```
**Expected:** `voiden-runner` sends the initial request, receives httpbin's
`401` + `WWW-Authenticate: Digest ...` challenge, computes the digest, and
re-sends — ending in `200`, `body.authenticated: true`.
**Actual:** Only one request is ever sent (visible in `--show-req` — no
`Authorization` header, and no second attempt logged). The run ends on the
`401` challenge itself. Bearer, Basic, and API-Key auth (see the same file's
other three sections) all work correctly headlessly — this looks isolated to
digest's extra round trip specifically.

## 6. MCP client and DeepWiki's `ask_question` tool (slow calls / SSE progress notifications)

**File:** N/A — deliberately left out of `mcp/deepwiki-tool-call.void` so the
fixture stays deterministic. `read_wiki_structure` and `read_wiki_contents`
(same file) both work correctly and quickly (~2–5s) against the same server.
**Repro:**
```bash
curl -s -X POST https://mcp.deepwiki.com/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ask_question","arguments":{"repoName":"anthropics/claude-code","question":"What CLI commands does this project expose?"}}}'
```
This curl succeeds in ~18s — DeepWiki streams `notifications/message` progress
events ("Processing query... (15s elapsed)") before the final `result`. The
equivalent `.void` `mcp-connection` / `mcpoperation` (`capabilityType: tool`,
`toolName: ask_question`) run through `voiden-runner run` fails after ~17s
with a bare `MCP request failed` and no response body — right around where the
real answer would have arrived. Unclear yet whether this is a hard client-side
timeout or the interim `notifications/message` events aren't handled — worth a
closer look from whoever owns the MCP client plugin, since any tool whose
backing service legitimately takes 15s+ will hit the same wall.

## 7. The `/tool` extension's CLI surface isn't in the published runner yet

**Affects:** `mcp/tool-decorated-request.void` (the `tool`/`toolparams`/
`toolverifies` blocks) — file is correctly authored and the underlying REST
request runs fine standalone, but nothing in `@voiden/runner@2.2.0` can
exercise the tool-serving behavior itself yet:
```
voiden-runner tool verify mcp/tool-decorated-request.void   # error: unknown command 'tool'
voiden-runner mcp serve .                                    # no such subcommand — `mcp` only has install/uninstall/status
```
The `voiden-mcp` skill (installed alongside `voiden`) documents `tool list`,
`tool verify`, and `mcp serve [--http]` in detail — these appear to be ahead
of what's actually published to npm. `mcp/tool-decorated-request.void`'s
`{{customer_name}}`/`{{customer_email}}` placeholders (normally filled in by
an MCP client via the `/tool` block's `binds` mechanism at call time) are
given plain fallback values in `.env` for now, purely so the underlying
request still runs in a blanket `voiden-runner run .` sweep — that is a
workaround, not a real exercise of the tool-serving path. Once `tool verify` /
`mcp serve` ship, swap this for the real thing and drop the `.env` fallback.

---

## Not a bug — worth noting for docs

- **`--env` accepting `.yaml` is inconsistent with the app's own env format.**
  `voiden-runner run --help` advertises `-e, --env <path>` as accepting ".env
  or .yaml", and the source (`packages/voiden-runner/src/envProfiles.ts` /
  `--profile`) clearly supports the app's own `.voiden/env-<profile>-{public,private}.yaml`
  tree — but that support isn't in this published CLI version (`--profile` is
  `error: unknown option`), and pointing `--env` straight at a
  `.voiden/env-public.yaml` file fails with `Malformed line 1 in .env file:
  missing "="` (it's trying to parse the YAML as a flat `.env` file). This
  repo ships a working `.env` (flat `KEY=value`) at the root for that reason —
  `.voiden/env-public.yaml` is kept too, since that's what the **Voiden app**
  itself reads via its Environment Editor, but the CLI/CI path currently needs
  the flat file. Once `--profile` (or real YAML parsing in `--env`) ships,
  this repo can drop the duplication.
