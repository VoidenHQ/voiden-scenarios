# Importers/

Source collections in four other tools' **native** formats — the same "Widget API" (4 operations: query params, path params, a JSON body, a multipart upload; bearer auth; scripting where the tool has it) written once per tool.

| Folder | Format | Notes |
|---|---|---|
| [`Postman/`](Postman/) | Postman v2.1 collection + environment | `pm.*` pre-request/test scripts |
| [`Insomnia/`](Insomnia/) | Insomnia v4 export (`resources[]`) | `insomnia.*` scripts; auth merges into headers, no dedicated auth block |
| [`OpenAPI/`](OpenAPI/) | OpenAPI 3.0 spec (YAML) | No scripting — the format doesn't have any |
| [`Bruno/`](Bruno/) | Bruno OpenCollection export (single YAML file) | `bru`/`req`/`res`/`test`/`expect` scripts |

These are **not** meant to be clicked through the Voiden app's own one-click **Import into Voiden** button (though they'd work fine that way too, each producing its usual one-file-per-request output). They exist as raw source fixtures for the *other* documented path: an agent generating `.void` files directly from a raw collection/spec, following each importer's mapping table in the `voiden` skill (Postman, OpenAPI, Bruno) or the plugin's own `src/skill.md` (Insomnia).

All four target `https://httpbin.org` for real, live verification.

A side-by-side folder of the four tools' *generated* `.void` output (`imported/`) used to live alongside this one — it was removed in the September 16 cleanup (see KNOWN-ISSUES.md #1, #6, and the Insomnia-skill note for what it found while it existed). These `Importers/` source fixtures themselves are untouched and still valid for regenerating that comparison by hand if needed.
