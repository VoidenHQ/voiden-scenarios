# voiden-scenarios

A `.void` file for every core Voiden feature, each one making **real requests
against real public APIs** — no mocks — so running this repo is an actual
end-to-end check of Voiden itself, not just a syntax sample. Built to double
as:

1. A CI fixture — `.github/workflows/validate-void.yml` runs the whole repo
   with `@voiden/runner` on every push/PR, the way a team adopting Voiden
   would wire it into their own CI.
2. A living reference — one file per feature area, meant to be opened in the
   Voiden app and read, not just executed.
3. A regression net for the beta — see **[KNOWN-ISSUES.md](KNOWN-ISSUES.md)**.
   Building and actually *running* this repo (not just authoring it) surfaced
   several real gaps in the currently-published `@voiden/runner@2.2.0`,
   including one — assertion failures not failing the CI run — that directly
   undermines the "validate void files in CI" pitch until it's fixed.

## Layout

| Folder | Covers |
|---|---|
| [`rest/`](rest/) | HTTP methods, headers/query/path tables, all body types, options-table, cookies-table |
| [`auth/`](auth/) | Bearer, Basic, API Key, Digest |
| [`crud-and-chaining/`](crud-and-chaining/) | Full CRUD + `runtime-variables` chaining across sections |
| [`graphql/`](graphql/) | Queries with and without variables |
| [`scripting/`](scripting/) | `pre_script`/`post_script` in all three languages (JS, Python, Shell) |
| [`assertions/`](assertions/) | Every `assertions-table` operator |
| [`mcp/`](mcp/) | MCP client (`mcp-connection`) against a real third-party server, plus a `/tool`-decorated request for `voiden-mcp-tool` |
| [`multi-section/`](multi-section/) | `request-separator` sectioning + cross-section chaining, isolated from any specific resource |

Not covered yet (next pass, see the bottom of this file): importers (Postman/
Bruno/Insomnia/HAR/OpenAPI), sockets/gRPC, the stitch runner, and faker.

## Running it

```bash
npm install -g @voiden/runner
voiden-runner run . --env .env --no-session
```

Everything targets public, no-signup APIs — [httpbin.org](https://httpbin.org)
(REST mechanics), [reqres.in](https://reqres.in) (CRUD), the
[Countries GraphQL API](https://countries.trevorblades.com) (GraphQL), and
[DeepWiki's public MCP server](https://mcp.deepwiki.com/mcp) (MCP) — so this
runs the same way for anyone, no credentials needed.

**Two env files, on purpose:**
- **`.env`** (repo root) — flat `KEY=value`, what `@voiden/runner@2.2.0`
  actually reads today via `--env`.
- **`.voiden/env-public.yaml`** — the same variables in the format the
  **Voiden app's** own Environment Editor reads. The CLI doesn't consume this
  one yet (see KNOWN-ISSUES.md's last section) — keep both in sync until it
  does.

Open the repo in the Voiden app directly to browse/run files interactively
instead — no separate setup needed there, the app reads `.voiden/env-public.yaml`.

## Known issues

Several `assertions-table` rows in this repo are intentionally `disabled: true`
with an inline comment pointing here, rather than deleted — they document a
real, reproduced gap in `@voiden/runner@2.2.0`, not a mistake in the fixture.
Full repro steps for each: **[KNOWN-ISSUES.md](KNOWN-ISSUES.md)**. Re-enable
each row once its underlying issue is fixed.

## Contributing more scenarios

Each `.void` file was authored by hand against the `voiden` skill (block
syntax) — see that skill (or the app's own slash commands) for the full
block reference. Conventions used throughout this repo:
- One file per feature area, multiple `request-separator` sections per file
  where it reads naturally as a group.
- Every section ends with an `assertions-table` (or, in `scripting/`,
  `voiden.assert` calls) that checks something real about the response —
  never just "status is 200" when a more specific check is available.
- Fresh UUID v4 for every block `uid` — never copy one from another file.
- Prefer a public, no-auth API over a mock wherever one exists for the
  scenario, so a passing run is a real, external signal.
