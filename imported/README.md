# imported/

The same 4-operation Widget API (`importers/*`), hand-generated into `.void` form once per source tool, following each importer's documented mapping table. All 16 requests (4 files × 4 operations) run against real `httpbin.org` endpoints and pass — verified with:

```bash
voiden-runner run imported/ --profile --no-session
```

| File | From | Auth shape | Scripting |
|---|---|---|---|
| [`from-postman/widgets.void`](from-postman/widgets.void) | `importers/postman/` | `auth` block | `pre_script`/`post_script`, translated from `pm.*` |
| [`from-insomnia/widgets.void`](from-insomnia/widgets.void) | `importers/insomnia/` | **`headers-table`, not `auth`** — see its header note | `pre_script`/`post_script`, translated from `insomnia.*` |
| [`from-openapi/widgets.void`](from-openapi/widgets.void) | `importers/openapi/` | `auth` block | **none** — OpenAPI has no scripting |
| [`from-bruno/widgets.void`](from-bruno/widgets.void) | `importers/bruno/` | `auth` block | `pre_script`/`post_script`, translated from `bru`/`req`/`res`/`test`/`expect` |

## How similar is "almost similar," concretely

The "Create Widget" section's translated script ends up **byte-for-byte identical** across Postman, Insomnia, and Bruno:

```javascript
voiden.assert(voiden.response.status, "==", 200, "Create widget succeeds");
voiden.assert((voiden.response.body.json)["name"], "truthy", "", "Create widget succeeds");
```

Three different source tools, three different scripting APIs (`pm.*`, `insomnia.*`, `bru`/`test`/`expect`), one identical result — because each one expresses the same check, and each importer's translation table converges on the same `voiden.*` primitives. That convergence is the actual point of this folder, not just a coincidence worth mentioning.

The one real, **intentional** structural difference is Insomnia's: its own format has no separate auth block, so every section's bearer token lands in a `headers-table` row instead of an `auth` block — each file's own header note explains it, and the importer skill is explicit that this shouldn't be "fixed" to match the others.

## A real bug this surfaced, not just a demo

Every `auth`-block section here (Postman, OpenAPI, Bruno — 9 sections total) carries a **disabled** assertion:

```
row: ["Bearer token was actually sent (known beta bug)", "requestHeader.Authorization", "equals", "Bearer widget-demo-token"]
```

`--show-req` confirms why it's disabled: `Authorization: Bearer ` — empty, every time — because of `KNOWN-ISSUES.md` #1 (bearer/basic/apiKey auth block values not being read, beta-only regression). `httpbin.org` doesn't enforce auth on any of these routes, so the requests still return `200` and every *other* assertion still passes — meaning this bug would ship completely invisibly in a real generated collection unless someone thought to check the actual request headers, exactly what these disabled rows are for.

The Insomnia file's equivalent rows are the mirror image — **enabled**, and passing, since its headers-table-based auth isn't affected by the same bug:

```
row: ["Bearer token actually sent (unaffected by KNOWN-ISSUES.md #1 — headers-table, not auth block)", "requestHeader.Authorization", "equals", "Bearer widget-demo-token"]
```

## One more thing worth a closer look, not yet confirmed either way

`from-openapi/widgets.void` includes an `openapispecLink` block per section (per the OpenAPI importer skill: "Voiden's own post-response pipeline... re-fetches the spec and validates the actual HTTP response... against what that operation documents"). Running the file shows no sign of that validation happening — no extra log output, no extra field in `--output-json`'s `reportEntries[]` — but there's also no error, so this is logged here as **unclear, not confirmed broken**: it might be an app-UI-only feature not wired into the headless `voiden-runner` CLI, or it might validate silently on success with nothing to show. Worth someone with access to the plugin's source confirming either way.
