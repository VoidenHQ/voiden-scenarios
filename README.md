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
   several real gaps, including one — assertion failures not failing the CI
   run — that directly undermines the "validate void files in CI" pitch until
   it's fixed, and one outright regression between stable and beta (auth
   credentials not being read).

## Layout

| Folder | Covers |
|---|---|
| [`rest/`](rest/) | HTTP methods, headers/query/path tables, all body types, options-table, cookies-table |
| [`crud-and-chaining/`](crud-and-chaining/) | Full CRUD + `runtime-variables` chaining across sections |
| [`graphql/`](graphql/) | Queries with and without variables |
| [`scripting/`](scripting/) | `pre_script`/`post_script` in all three languages (JS, Python, Shell) |
| [`assertions/`](assertions/) | Every `assertions-table` operator |
| [`mcp/`](mcp/) | MCP client (`mcp-connection`) against a real third-party server, plus a `/tool`-decorated request for `voiden-mcp-tool` |
| [`multi-section/`](multi-section/) | `request-separator` sectioning + cross-section chaining, isolated from any specific resource |
| [`known-broken-under-beta/`](known-broken-under-beta/) | Auth types (Bearer, Basic, API Key, Digest) — **excluded from CI on purpose**, see that folder's file header and KNOWN-ISSUES.md #1/#2 |
| [`importers/`](importers/) | The *same* Widget API (query/path params, JSON body, multipart upload, bearer auth, scripting) hand-written once per tool — Postman, Insomnia, OpenAPI, Bruno — in each one's own native format |
| [`imported/`](imported/) | Those four native collections generated into `.void` form, one folder per source tool, so you can compare what each importer's mapping actually produces side by side — see `imported/README.md` |

Not covered yet (next pass): HAR importer, sockets/gRPC, the stitch runner,
and faker.

## Running it

This repo targets the **beta** channel (`@voiden/runner@beta`, currently
`2.3.0-beta.19` — the same version stamped in every file's frontmatter here),
not `latest`, since that's what surfaced/fixed several of the issues in
KNOWN-ISSUES.md. Switch to `@voiden/runner` (no `@beta`) once the team is off
beta for good.

```bash
npm install -g @voiden/runner@beta
voiden-runner plugin update --all   # always do this right after installing/switching — see KNOWN-ISSUES.md
voiden-runner run rest/ assertions/ crud-and-chaining/ graphql/ mcp/ multi-section/ scripting/ imported/ --profile --no-session
```

Everything targets public, no-signup APIs — [httpbin.org](https://httpbin.org)
(REST mechanics), [reqres.in](https://reqres.in) (CRUD), the
[Countries GraphQL API](https://countries.trevorblades.com) (GraphQL), and
[DeepWiki's public MCP server](https://mcp.deepwiki.com/mcp) (MCP) — so this
runs the same way for anyone, no credentials needed.

**`.voiden/env-public.yaml`** is the source of truth — it's what `--profile`
reads, and it's what the **Voiden app's** own Environment Editor reads too, so
one file covers both. A flat **`.env`** is also kept at the repo root, with
the same values, purely as a fallback for anyone still running the `latest`
(stable) channel, where `--profile` doesn't exist yet — use `--env .env` there
instead of `--profile`. Keep both in sync if you add a variable; drop `.env`
entirely once the team is fully off stable.

Open the repo in the Voiden app directly to browse/run files interactively
instead — no separate setup needed there, the app reads `.voiden/env-public.yaml`.

**Heads up on `crud-and-chaining/` and `mcp/tool-decorated-request.void`:**
both hit reqres.in, whose anonymous tier is capped at 40 requests/day per IP —
easy to exhaust during active local testing (it happened while building this
repo). A `429 rate_limit_exceeded` from those two files specifically is
almost always this, not a real failure — see KNOWN-ISSUES.md's last section.

## Known issues

Several `assertions-table` rows in this repo are intentionally `disabled: true`
with an inline comment pointing here, rather than deleted — they document a
real, reproduced gap, not a mistake in the fixture. One whole file
(`known-broken-under-beta/auth-types.void`) is excluded from CI entirely for
the same reason. Full repro steps for each, plus which are stable-only,
beta-only, or both: **[KNOWN-ISSUES.md](KNOWN-ISSUES.md)**. Re-enable a row
(or move the file back) once its underlying issue is fixed.

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
