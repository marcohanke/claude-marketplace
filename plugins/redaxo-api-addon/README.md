# redaxo-api-addon

Claude Code support for the [FriendsOfRedaxo/api](https://github.com/FriendsOfRedaxo/api) addon — a Bearer-token + backend-session REST API for REDAXO. Use it to create/read/modify articles, categories, slices, modules, templates, languages, media (incl. multipart upload), users, roles, and metainfo over HTTP.

Tracks the addon's `1.2+` release.

## Skills

- **api-routes** – Bearer + Backend-Session auth, the full route table (Structure, Templates, Modules, Clangs, Media, Users/Roles, Metainfo, Backend-mirror under `/api/backend/...`), the unified `{data, meta}` list format, the slice POST schema, OpenAPI spec, and a diagnostic flowchart for 401 / 404 / 405 / 500 responses.
- **api-extending** – building your own `RoutePackage` to expose custom endpoints, the `Body`/`query` schema keys, scope registration, the JsonResponse convention, request-context caveats (auth runs in frontend context, `isBackend()` returns `false`, PRE-EPs that call `rex::requireUser()` fail under Bearer), and the addon's exact-mirror-of-core convention.

## Install

```bash
/plugin install redaxo-api-addon@redaxo-marketplace
```

The skills assume the `api` addon (v1.2 or later) is installed and either a token has been created in **AddOns → API → Token** or the caller is logged into the REDAXO backend (for `/api/backend/...` routes).
