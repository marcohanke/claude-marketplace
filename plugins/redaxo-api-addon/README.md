# redaxo-api-addon

Claude Code support for the [FriendsOfRedaxo/api](https://github.com/FriendsOfRedaxo/api) addon — a Bearer-token-based REST API for REDAXO. Use it to create/read/modify articles, categories, slices, modules, templates, languages, media, and users over HTTP.

## Skills

- **api-routes** – authentication, the route table (incl. trailing-slash gotchas), the slice POST schema, OpenAPI spec, and a diagnostic flowchart for 401 / 404 / 500 responses
- **api-extending** – building your own `RoutePackage` to expose custom endpoints, scope registration, and request-context caveats

## Install

```bash
/plugin install redaxo-api-addon@redaxo-marketplace
```

The skills assume the `api` addon is installed and a token has been created in **AddOns → API → Token**.
