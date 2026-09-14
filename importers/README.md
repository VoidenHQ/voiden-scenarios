# importers/

Source collections in four other tools' **native** formats — the same "Widget API" (4 operations: query params, path params, a JSON body, a multipart upload; bearer auth; scripting where the tool has it) written once per tool, so `imported/` can show what generating directly from each one actually produces.

| Folder | Format | Notes |
|---|---|---|
| [`postman/`](postman/) | Postman v2.1 collection + environment | `pm.*` pre-request/test scripts |
| [`insomnia/`](insomnia/) | Insomnia v4 export (`resources[]`) | `insomnia.*` scripts; auth merges into headers, no dedicated auth block |
| [`openapi/`](openapi/) | OpenAPI 3.0 spec (YAML) | No scripting — the format doesn't have any |
| [`bruno/`](bruno/) | Bruno OpenCollection export (single YAML file) | `bru`/`req`/`res`/`test`/`expect` scripts |

These are **not** meant to be clicked through the Voiden app's own one-click **Import into Voiden** button (though they'd work fine that way too, each producing its usual one-file-per-request output). They exist so `imported/*/widgets.void` can demonstrate the *other* documented path: an agent generating `.void` files directly from a raw collection/spec, following each importer's mapping table in the `voiden` skill (Postman, OpenAPI, Bruno) or the plugin's own `src/skill.md` (Insomnia — see the note in `imported/from-insomnia/widgets.void`'s header about why).

All four target `https://httpbin.org` for real, live verification — see `imported/README.md` for how the generated files actually ran.
