# Testing the `voiden` Skill — Prompt Library

A prompt per generation scenario the `voiden` skill (plus its extension sections and `voiden-mcp`) documents, meant to be pasted one at a time into a fresh Claude Code / Claude session with the skill installed. Not part of CI — this tests the **skill's own output quality**, not a shipped artifact. For each prompt: run it, then check the output against the "Look for" line before moving on.

How to use this file:
1. Pick a prompt, paste it verbatim (or adapted to a scratch project).
2. Check the generated `.void` file against "Look for."
3. If the app is open, actually run the result and confirm it behaves — a `.void` file that *looks* right but doesn't run is still a skill failure.
4. Note anything off in `KNOWN-ISSUES.md` the same way the rest of this repo does — file, repro, expected vs. actual.

---

## Base skill — request construction

**1. curl → `.void`**
> Convert this into a Voiden request:
> `curl -X POST https://httpbin.org/post -H "Content-Type: application/json" -H "Authorization: Bearer abc123" -d '{"name":"Ada","role":"admin"}'`

Look for: `request` (POST), `headers-table` (Content-Type), `auth` block (`authType: bearer`, not a raw header — the skill maps `Authorization: Bearer` to a real auth block), `json_body` with the exact payload.

**2. Multi-section CRUD with chaining**
> Generate a multi-section .void file for a "Products" resource: Create, List, Get, Update, Delete, against `https://httpbin.org` (not a real backend — just exercise the shape). Chain the created id from Create into the later sections.

Look for: one `request-separator` per operation, correct block order inside each, a `runtime-variables` (or `post_script`) capture in Create, `{{process.*}}` (not bare `{{...}}`) used in every later section that needs it, separator `label`s naming each operation (no redundant `## Heading`s).

**3. Path + query + headers together**
> Add a request for `GET /orders/{orderId}/items?status=shipped&limit=25` with a custom `X-Tenant-Id` header, base URL `{{BASE_URL}}`.

Look for: `path-table` row for `orderId` (URL keeps `{orderId}`, not `:orderId`), `query-table` rows for `status`/`limit`, `headers-table` row for `X-Tenant-Id`, 3-column rows (Key/Value/Description) since these are freshly authored.

**4. Every assertion operator on one response**
> Write an assertions-table for a GET response that checks: status is 200, the body has an `id` field, `email` contains `@`, `role` is exactly `"admin"`, response time is under 800ms, and a `X-RateLimit-Remaining` header exists.

Look for: correct operator per check (`equals`, `exists`, `contains`, `less-than`, etc.), correct field-path prefixes (`status`, `body.x`, `responseTime`, `headers.x-ratelimit-remaining`).

---

## Auth

**5. One prompt per auth type** — run all of these against the same base request shape and confirm each produces the right `authType` + required rows (cross-check against the base skill's Auth Field Reference table):
> Add bearer auth using `{{API_TOKEN}}`.
> Switch this to basic auth with `{{API_USERNAME}}` / `{{API_PASSWORD}}`.
> Switch this to an API key sent as a query param named `api_key`.
> Switch this to OAuth2 client_credentials, token URL `{{OAUTH_TOKEN_URL}}`, scope `read write`.
> Switch this to OAuth2 authorization_code with a callback URL and PKCE-relevant fields.
> Switch this to AWS Signature v4, region `us-east-1`, service `execute-api`.
> Switch this to Digest auth.

Look for (every time): exactly **one** `auth` block in the section — switching type edits it in place, never adds a second one. This is the single most common way an agent gets this wrong.

---

## Body types

**6. One prompt per body type**
> Give this request a JSON body with a nested object and an array.
> Same request, but XML body instead.
> Same request, but YAML body.
> Same request, but multipart form with two text fields and one file field.
> Same request, but application/x-www-form-urlencoded with three fields.

Look for: correct block type per case (`json_body`/`xml_body`/`yml_body`/`multipart-table`/`url-table`), and for the file field — the skill should **not** hand-craft a fake `fileLink` node; it should either say a real file needs attaching from the UI, or (per the importer skills' documented fallback) keep a given path as plain text.

---

## GraphQL

**7. Query + variables**
> Write a GraphQL request against `{{GRAPHQL_URL}}` for a query named `GetUser($id: ID!)` returning `id`, `name`, `email`, with variables `{"id": "42"}`.

Look for: `gqlquery` container with `gqlurl`/`gqlbody` children (not wrapped in a `request` block), separate `gqlvariables` block, `operationType: query`.

**8. Mutation**
> Same, but a mutation that creates a user and returns its id.

Look for: `operationType: mutation`, request body text actually says `mutation`.

---

## Scripting

**9. Same script, three languages**
> Write a pre_script that adds a header `X-Trace-Id` set to a random UUID, and a post_script that asserts status 200 and logs the response time. Give me the JavaScript version.
> Same thing in Python.
> Same thing in Shell.

Look for: correct `language` attr per block, correct API shape per language (JS/Python use `voiden.request.headers.push({...})`; Shell uses `voiden.request.headers.push "X-Trace-Id" "..."` — no `vd` alias in Shell), and that the Shell version doesn't invent a `vd.*` call anywhere.

**10. Full `voiden.assert` operator sweep in a script**
> Write a post_script (JavaScript) asserting: status equals 200, response time less than 1000ms, body.id is truthy, body.role contains "admin", and a header matches a UUID regex.

Look for: correct operator strings (`==`, `<`, `truthy`, `contains`, `matches`), regex passed as plain source text with no `/.../ ` delimiters.

---

## MCP

**11. MCP client from a pasted config**
> Here's an MCP server config, set up a connection and call its first tool:
> `{"mcpServers": {"weather": {"url": "https://example.com/mcp", "headers": {"Authorization": "Bearer {{WEATHER_TOKEN}}"}}}}`

Look for: `mcp-connection` with `mcpurl` filled from the config, a `headers-table` (or, per the actual mechanism, the config's headers applied) carrying the Authorization header, `mcpoperation` with a placeholder tool name for the user to fill in (the skill can't know the real tool name without connecting).

**12. Decorate an existing request as a tool**
> Decorate the Create Widget request as an agent-callable tool named `create_widget`, with `name` and `price` as agent-supplied parameters, and verify it against its own happy path.

Look for: `tool` container referencing the sibling request's `requestUid`, `toolparams` rows with `binds` matching real `{{token}}`s **already in the request** (not invented ones), `toolverifies` with a `happy-path` row pointing at the correct `sectionLabel`.

**13. A tool needing an environment-sourced param**
> This request uses `{{BASE_URL}}` — decorate it as a tool without exposing BASE_URL to the calling agent.

Look for: a `toolparams` row with `source: environment` for `BASE_URL` specifically (not `source: agent`, which is the default an agent might reach for without thinking) — this is exactly the gap found live in `imported/*/widgets.void` and `mcp/tool-decorated-request.void` while building this repo; a good prompt to catch a regression.

---

## Importers — generate directly from a raw collection/spec

**14. OpenAPI → grouped `.void`**
> Here's an OpenAPI spec with 5 operations across `/users` and `/users/{id}` — generate Voiden requests from it.
> *(paste `importers/openapi/widget-api.openapi.yaml` or any real spec)*

Look for: **one file per resource**, not per operation — sections in CRUD order, `{{BASE_URL}}` centralized as an env var (not the literal server URL baked into every section), an `auth` block derived from `securitySchemes`, an `openapispecLink` block per section.

**15. Postman → grouped `.void` with script translation**
> Convert this Postman collection into Voiden requests.
> *(paste `importers/postman/widget-api.postman_collection.json`)*

Look for: one file per folder/resource, `pm.test`/`pm.expect` scripts translated live where every line is safe (per the translation table), left fully commented (not half-translated) where anything isn't.

**16. Bruno OpenCollection export**
> Convert this Bruno collection export into Voiden requests.
> *(paste `importers/bruno/widget-api.bruno.yaml`)*

Look for: correct `opencollection` marker detection, `runtime.scripts[]`/`runtime.assertions[]` mapped correctly.

**17. Insomnia export**
> Convert this Insomnia export into Voiden requests.
> *(paste `importers/insomnia/widget-api.insomnia.json`)*

Look for: auth merged into `headers-table` (**not** turned into an `auth` block — a very easy, very wrong "fix" for an agent to make), Insomnia's `{{ _.VAR }}` template tags correctly rewritten to Voiden's `{{VAR}}`.

**18. A classic Bruno folder (the sibling-discovery trap)**
> Here's one `.bru` file from a Bruno collection — convert it. *(paste a single `create-x.bru`-shaped file, don't mention siblings)*

Look for: does the agent **stop and ask to see the containing folder** (or explicitly note it can't safely group without seeing siblings), rather than silently defaulting to one file per request? This is the specific, documented failure mode the Bruno skill calls out — the best test of whether the skill's warning actually lands.

---

## Reuse

**19. `linkedBlock`**
> I have a shared headers block in `/shared/base-headers.void` (uid `825cb614-...`) — reuse it in this new request instead of duplicating the headers.

Look for: fresh `uid` on the `linkedBlock` itself (never reused from the target), correct `blockUid`/`originalFile`, and — if you first point it at a file that turns out to itself be a `linkedBlock` — confirm it follows the chain to the real origin rather than pointing at the pointer.

**20. `linkedFile`**
> Import the "Login" section from `/auth/session.void` into this file as a reusable section.

Look for: a `request-separator` immediately before the `linkedFile`, correct `sectionUid` (the target separator's own uid, not guessed), placed at the top level (never nested inside a `request`).

---

## Environment variables

**21. From scratch**
> Set up environment variables for `dev` and `staging`, where `staging` has a `eu` sub-environment overriding just the region.

Look for: correct `.voiden/env-*.yaml` tree shape, `children` nesting for `staging.eu`, no literal dots typed into node names.

**22. Runtime capture**
> After this login request, capture the access token and expose it as `{{process.access_token}}` for later requests.

Look for: a `runtime-variables` block with the right `{{$res.body...}}` capture expression, and every later reference using the `process.` prefix — never a bare `{{access_token}}`.
