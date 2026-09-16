# voiden-scenarios

A `.void` file for every core Voiden feature, each one making **real requests
against real public APIs** — no mocks — so running this repo is an actual
end-to-end check of Voiden itself, not just a syntax sample. Built to double
as:

1. A CI fixture *and* a local one — `.github/workflows/validate-void.yml`
   runs `Automated Tests/` with `@voiden/runner` on every push/PR, the way a
   team adopting Voiden would wire it into their own CI, but the exact same
   folders run identically from your own terminal (see "Running it" below)
   or opened directly in the Voiden app — CI isn't required to exercise any
   of this.
2. A living reference — one file per feature area, meant to be opened in the
   Voiden app and read, not just executed.
3. A regression net for the beta — see **[KNOWN-ISSUES.md](KNOWN-ISSUES.md)**.
   Building and actually *running* this repo (not just authoring it) surfaced
   several real gaps, including one — assertion failures not failing the CI
   run — that directly undermines the "validate void files in CI" pitch until
   it's fixed, and one outright regression between stable and beta (auth
   credentials not being read).

## Layout

Two top-level groups, so it's obvious at a glance which files are fully
scripted (no human needed to complete a run) versus which genuinely require
one:

- **[`Automated Tests/`](Automated%20Tests/)** — fully scripted, assertion-checked, no human interaction needed to complete a run. That's what makes it CI-safe: `validate-void.yml` runs every folder here on every push/PR (except `Known Broken (Beta)/`, see below) — but **it's just as runnable from your own machine**, with the exact same `voiden-runner run` commands (see "Running it" below), or by opening any file in the Voiden app and hitting Run. CI running it isn't what makes it "automated" — being fully scripted is; CI is just one more place that happens to run it unattended.
- **[`Manual Testing/`](Manual%20Testing/)** — the opposite: everything that genuinely needs a human, a browser, real credentials, or a connected agent to complete. **Not run by CI at all** — CI simply can't do these unattended, but you still run them yourself locally the same way.

### `Automated Tests/`

| Folder | Covers |
|---|---|
| [`REST/`](Automated%20Tests/REST/) | HTTP methods, headers/query/path tables, all body types, options-table, cookies-table |
| [`CRUD and Chaining/`](Automated%20Tests/CRUD%20and%20Chaining/) | Full CRUD + `runtime-variables` chaining across sections |
| [`GraphQL/`](Automated%20Tests/GraphQL/) | Queries with and without variables |
| [`Scripting/`](Automated%20Tests/Scripting/) | `pre_script`/`post_script` in all three languages (JS, Python, Shell) |
| [`Assertions/`](Automated%20Tests/Assertions/) | Every `assertions-table` operator |
| [`MCP/`](Automated%20Tests/MCP/) | MCP client (`mcp-connection`) against a real third-party server, plus a `/tool`-decorated request for `voiden-mcp-tool` |
| [`Multi Section/`](Automated%20Tests/Multi%20Section/) | `request-separator` sectioning + cross-section chaining, isolated from any specific resource |
| [`Importers/`](Automated%20Tests/Importers/) | The *same* Widget API (query/path params, JSON body, multipart upload, bearer auth, scripting) hand-written once per tool — Postman, Insomnia, OpenAPI, Bruno — in each one's own native format. Source fixtures only (not runnable `.void` files themselves), so excluded from the CI run command below |
| [`Known Broken (Beta)/`](Automated%20Tests/Known%20Broken%20%28Beta%29/) | Auth types (Bearer, Basic, API Key, Digest) — **excluded from CI on purpose**, see that folder's file header and KNOWN-ISSUES.md #1/#2 |

### `Manual Testing/`

Manual checklists, a deliberately-mixed-state `/tool` scenario, AI-skill test
prompts, and an OAuth fill-in-your-own-credentials template. See
[`Manual Testing/README.md`](Manual%20Testing/README.md).

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
voiden-runner run "Automated Tests/REST" "Automated Tests/Assertions" "Automated Tests/CRUD and Chaining" \
  "Automated Tests/GraphQL" "Automated Tests/MCP" "Automated Tests/Multi Section" "Automated Tests/Scripting" \
  --profile --no-session
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

**Heads up on `CRUD and Chaining/` and `MCP/Tool Decorated Request.void`:**
both hit reqres.in, whose anonymous tier is capped at 40 requests/day per IP —
easy to exhaust during active local testing (it happened while building this
repo). A `429 rate_limit_exceeded` from those two files specifically is
almost always this, not a real failure — see KNOWN-ISSUES.md's last section.

## Known issues

Several `assertions-table` rows in this repo are intentionally `disabled: true`
with an inline comment pointing here, rather than deleted — they document a
real, reproduced gap, not a mistake in the fixture. One whole file
(`Automated Tests/Known Broken (Beta)/Auth Types.void`) is excluded from CI
entirely for the same reason. Full repro steps for each, plus which are
stable-only, beta-only, or both: **[KNOWN-ISSUES.md](KNOWN-ISSUES.md)**.
Re-enable a row (or move the file back) once its underlying issue is fixed.

## Contributing more scenarios

Each `.void` file was authored by hand against the `voiden` skill (block
syntax) — see that skill (or the app's own slash commands) for the full
block reference. Conventions used throughout this repo:
- One file per feature area, multiple `request-separator` sections per file
  where it reads naturally as a group.
- Every section ends with an `assertions-table` (or, in `Scripting/`,
  `voiden.assert` calls) that checks something real about the response —
  never just "status is 200" when a more specific check is available.
- Fresh UUID v4 for every block `uid` — never copy one from another file.
- Prefer a public, no-auth API over a mock wherever one exists for the
  scenario, so a passing run is a real, external signal.
