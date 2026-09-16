# Manual Testing/

Manual, human-in-the-loop testing material — **not** part of `.github/workflows/validate-void.yml`, and not meant to be. Everything here either needs a real connected agent, a browser-based interactive flow, or credentials nobody but you has, none of which a CI runner can do unattended. The action runs what it can run automatically; this is what you run yourself, plus the checks the action can't do at all.

| File | What it's for |
|---|---|
| [`MCP Testing Checklist.md`](MCP%20Testing%20Checklist.md) | Step-by-step checklist for `voiden-mcp-client` (connecting to a server) and `voiden-mcp-tool` (serving requests as one) — CLI checks plus connecting a real agent |
| [`Tool Scenario/Widget Tools.void`](Tool%20Scenario/Widget%20Tools.void) | Five tools, five different verification states (verified / failing+withdraw / failing+degraded / unverified / auth-gated) — the concrete fixture the checklist above walks through |
| [`AI Skill Testing Prompts.md`](AI%20Skill%20Testing%20Prompts.md) | One prompt per generation scenario the `voiden` skill documents — paste into a fresh Claude session to test the skill's own output quality |
| [`OAuth Scenarios/OAuth Every Grant Type.void`](OAuth%20Scenarios/OAuth%20Every%20Grant%20Type.void) | Every OAuth grant type (OAuth2 × 4 + OAuth1), as both plain `auth` blocks and — for the headless-friendly ones — `/tool` decorations. A fill-in-your-own-credentials template, not runnable as-is |
| [`Video Script - MCP and AI Skill.md`](Video%20Script%20-%20MCP%20and%20AI%20Skill.md) | A ready-to-shoot demo script — Postman collection → AI-generated `.void` suite → `/tool` verification catching a real broken tool → a live connected agent calling it. Every command in it is real, already run against this repo |
| [`Fixes Log/`](Fixes%20Log/) | Dated running log of real bugs found and fixed while working on this repo/the plugins, one file per day |

## Before you start

- `voiden-runner plugin update --all` after any `@voiden/runner` version switch (see `KNOWN-ISSUES.md`'s opening note) — a stale plugin cache masquerades as a fresh regression otherwise.
- `mcp serve`'s `[path]` argument wants the **project root**, not a specific file — `voiden-runner mcp serve . --check --profile`, not `voiden-runner mcp serve "Manual Testing/Tool Scenario/Widget Tools.void" --check --profile` (the latter silently breaks `--profile`'s env resolution — a real gap, noted in the checklist).
- Real secrets (OAuth client secrets, anything under `Manual Testing/OAuth Scenarios/`) go in `.voiden/env-private.yaml`, never `.voiden/env-public.yaml` or `.env` — both of those are git-tracked.
