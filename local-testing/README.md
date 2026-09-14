# local-testing/

Manual, human-in-the-loop testing material — **not** part of `.github/workflows/validate-void.yml`, and not meant to be. Everything here either needs a real connected agent, a browser-based interactive flow, or credentials nobody but you has, none of which a CI runner can do unattended. The action runs what it can run automatically; this is what you run yourself, plus the checks the action can't do at all.

| File | What it's for |
|---|---|
| [`mcp-testing-checklist.md`](mcp-testing-checklist.md) | Step-by-step checklist for `voiden-mcp-client` (connecting to a server) and `voiden-mcp-tool` (serving requests as one) — CLI checks plus connecting a real agent |
| [`tool-scenario/widget-tools.void`](tool-scenario/widget-tools.void) | Five tools, five different verification states (verified / failing+withdraw / failing+degraded / unverified / auth-gated) — the concrete fixture the checklist above walks through |
| [`ai-skill-testing-prompts.md`](ai-skill-testing-prompts.md) | One prompt per generation scenario the `voiden` skill documents — paste into a fresh Claude session to test the skill's own output quality |
| [`oauth-scenarios/oauth-every-grant-type.void`](oauth-scenarios/oauth-every-grant-type.void) | Every OAuth grant type (OAuth2 × 4 + OAuth1), as both plain `auth` blocks and — for the headless-friendly ones — `/tool` decorations. A fill-in-your-own-credentials template, not runnable as-is |
| [`video-script-mcp-and-ai-skill.md`](video-script-mcp-and-ai-skill.md) | A ready-to-shoot demo script — Postman collection → AI-generated `.void` suite → `/tool` verification catching a real broken tool → a live connected agent calling it. Every command in it is real, already run against this repo |

## Before you start

- `voiden-runner plugin update --all` after any `@voiden/runner` version switch (see `KNOWN-ISSUES.md`'s opening note) — a stale plugin cache masquerades as a fresh regression otherwise.
- `mcp serve`'s `[path]` argument wants the **project root**, not a specific file — `voiden-runner mcp serve . --check --profile`, not `voiden-runner mcp serve local-testing/tool-scenario/widget-tools.void --check --profile` (the latter silently breaks `--profile`'s env resolution — a real gap, noted in the checklist).
- Real secrets (OAuth client secrets, anything under `local-testing/oauth-scenarios/`) go in `.voiden/env-private.yaml`, never `.voiden/env-public.yaml` or `.env` — both of those are git-tracked.
