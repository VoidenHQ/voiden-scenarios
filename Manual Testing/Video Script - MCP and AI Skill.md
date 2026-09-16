# Video Script — "From Collection to AI-Callable Tool"

**Covers:** the `voiden` AI skill (generating `.void` files) + MCP, both directions (Voiden as MCP client, Voiden as MCP server via `voiden-mcp-tool`).
**Length:** ~7–8 minutes.
**Format:** screen recording, single take per scene, voiceover either live or dubbed after.

## The story, in one line

*A developer inherits an old Postman collection for a "Widget API," gets Claude to turn it into a real Voiden test suite in seconds, decorates the important calls as AI-callable tools, watches Voiden catch a genuinely broken one before it ever reaches an agent — then connects Claude Code live and asks it to create a widget for real.*

That's the actual demo. It's not contrived — it's the real workflow this repo's own `Automated Tests/Importers/` → generated `.void` output → `Manual Testing/Tool Scenario/` pipeline walked through while building it, just narrated.

## Before you hit record

- [ ] `voiden-scenarios` repo open in the Voiden app, and in a terminal (`cd` into it)
- [ ] `@voiden/runner@beta` installed, `plugin update --all` already run (don't do this on camera, it's boring)
- [ ] A fresh Claude Code session in this repo, with the `voiden` and `voiden-mcp` skills available
- [ ] `voiden-runner mcp install` already run once beforehand so Claude Code is registered — or plan to show that as its own quick beat (Scene 5)
- [ ] Close everything unrelated. One terminal, one editor/app window, one Claude Code pane.

---

## Scene 1 — The problem (0:00–0:30)

**On screen:** talking-head or voiceover-over-blank-editor. Maybe `Automated Tests/Importers/Postman/Widget API.postman_collection.json` open, unscrolled.

**Narration:**
> "Say you've got an old Postman collection for an API — nothing fancy, a few endpoints for managing widgets. You want two things: a real test suite you can trust, and for your AI coding assistant to be able to *actually call* this API while it's helping you build — not just read about it. Here's the whole path, start to finish."

---

## Scene 2 — AI skill: collection in, real `.void` file out (0:30–2:00)

**On screen:** Claude Code, this repo open.

**Type this prompt** (or read close to it):
> "Convert `Automated Tests/Importers/Postman/Widget API.postman_collection.json` into Voiden requests."

**While it runs, narrate:**
> "This is the `voiden` skill — Claude reads the raw Postman JSON and generates real `.void` files from the mapping rules it's been taught, not a canned template. Watch what it does with the tricky parts."

**Cut to the generated output** (have Claude write it to somewhere like `Automated Tests/Importers/Postman/Widget API.void` beforehand if the live generation takes too long to fully show):
- Point at the **auth block** — "bearer token, mapped straight from the collection's own auth config."
- Point at the **pre_script / post_script** — "this collection had a real pre-request script setting a header, and a real test script. Claude translated both into Voiden's own scripting API — line by line, only where it was safe to."
- Point at the **multipart upload section** — "and where the source had a file field with nothing to actually attach, it didn't fake one — it left an honest placeholder."

**Beat — run it:**
```
voiden-runner run "Automated Tests/Importers/Postman/Widget API.void" --profile --no-session
```
**Narration over the green output:**
> "Real requests, against a real API. Not a mock — this call actually left our machine."

---

## Scene 3 — Making it trustworthy: assertions + chaining (2:00–3:00)

**On screen:** `Automated Tests/CRUD and Chaining/Users CRUD.void` or `Automated Tests/REST/Headers Query Path.void` — pick whichever reads cleanest on screen.

**Narration:**
> "One more AI-skill beat, quickly: chaining. Ask it to capture a value from one response and use it two sections later."

**Prompt:**
> "After the create request, capture the new id and use it in the get/update/delete requests below."

**Point at:** the `runtime-variables` block, and `{{process.user_id}}` two sections down.

> "Notice the `process.` prefix — that's not decoration, a bare `{{user_id}}` silently never resolves. Small thing, easy to get wrong by hand, exactly the kind of detail worth having an AI that actually knows the format handle for you."

---

## Scene 4 — MCP Tool: turning a request into something an agent can call (3:00–5:00)

**This is the core of the video.** On screen: `Manual Testing/Tool Scenario/Widget Tools.void`.

**Narration:**
> "Now the part that actually matters: I don't want my agent calling *every* request in this project — I want to hand-pick a few, and I want Voiden to prove they work before an agent ever sees them. Here's five requests, each decorated as a tool, each in a different state on purpose."

**Run this on screen — the money shot:**
```
voiden-runner tool verify "Manual Testing/Tool Scenario/Widget Tools.void" --profile
```

**Let the real output play out**, then narrate over the result (it looks like this — already captured, this is real):
```
✓ verified    get_widget
✗ failing     delete_widget       (contract-failure)
✗ failing     archive_widget      (contract-failure)
○ unverified  ping_widgets_api
✗ failing     admin_purge_widgets (auth-failure — dependent check skipped)
```

> "`get_widget` genuinely works — verified. `delete_widget` is broken — I told Voiden to withdraw it on failure, so an agent will never even know it exists. `archive_widget` is broken too, but I marked it 'advertise degraded' instead — it still shows up, just with a warning in its description. And `admin_purge_widgets` — its own logic is actually fine, but the auth check *in front of it* failed, so Voiden correctly reports it as an auth problem, not a broken tool, and never even runs the part underneath."

**Then:**
```
voiden-runner mcp serve . --check --profile
```
> "Same story, from the serving side: withdrawn, served-but-degraded, served-unverified, withdrawn. This is what actually gets exposed."

---

## Scene 5 — Connect a real agent, and ask it to do something (5:00–7:00)

**On screen:** a fresh Claude Code session (or switch panes if already running), this project registered via `voiden-runner mcp install`.

**Narration:**
> "So let's connect for real."

**If not already installed, show it live — it's fast and worth 10 seconds on camera:**
```
voiden-runner mcp install --claude
```

**Then, in the Claude Code chat:**
> "What tools do you have from the widgets project?"

**Let Claude list them.** Narrate over the list:
> "Get widget, archive widget, ping — and that's it. No delete, no admin purge. It genuinely cannot see them."

**Then, the payoff prompt:**
> "Get widget widget-7 for me."

**Let it actually call the tool live**, show the real response coming back.

> "That's a live HTTP call, made by Claude, through Voiden, proven to work *before* Claude ever knew it existed."

---

## Scene 6 — Bonus beat: Voiden as an MCP *client* (7:00–7:45)

**On screen:** `Automated Tests/MCP/DeepWiki Tool Call.void`.

**Narration:**
> "One more direction, quickly — Voiden isn't just something an AI calls, it can call *other* MCP servers itself. Same file format, same `Run`."

**Run it, show the response:**
```
voiden-runner run "Automated Tests/MCP/DeepWiki Tool Call.void" --profile --no-session
```

> "That's Voiden itself talking to a real third-party MCP server and pulling back live documentation — no custom integration code, just a `.void` file."

---

## Scene 7 — Close (7:45–8:00)

**On screen:** back to the terminal or the app's file tree, everything built visible.

**Narration:**
> "One old Postman collection. An AI that understood the format well enough to translate it faithfully, including the scripts. A verification step that caught a real bug before it ever reached an agent. And a live agent, actually calling the API. That's the whole loop."

---

## Cutting notes

- Scenes 2 and 4 are the ones worth the most editing polish — Scene 4's terminal output is the single most important shot in the video, since it's the concrete proof of the safety story. Don't cut away from it too fast.
- If recording in one continuous take is too long, Scenes 1–3 (AI skill) and Scenes 4–7 (MCP) split cleanly into two shorter videos — "Generating tests with AI" and "Making your API AI-callable, safely" — if a shorter-form cut is wanted instead of one long video.
- Everything in Scenes 2–6 is a real, already-verified command against this exact repo — rehearse once, but nothing here is scripted output or fake data.
