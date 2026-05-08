---
name: api-routes
description: Calling the FriendsOfRedaxo/api REST endpoints – Bearer auth, route table for articles/categories/slices/modules/templates/languages/media/users, the slice POST schema, OpenAPI spec, and a 401/404/500 diagnostic flow. Covers the known trailing-slash quirks, the two different 401 messages and what they mean, the LIMIT-binding bug, the article list `name`-filter bug, and yrewrite cache invalidation. Use when the user calls the api addon over HTTP, hits a confusing 401/404, syncs articles between systems via this API, or builds tooling around it.
---

# REDAXO `api` Addon – Calling the API

The `api` addon (Repo: `FriendsOfRedaxo/api`, Vendor-NS: `FriendsOfRedaxo\Api`) exposes a RESTful API for a REDAXO backend. Bearer-token auth, Symfony `UrlMatcher` routing, mounted at `/api/...` via `YREWRITE_PREPARE`.

## Activation & auth

- Tokens are created in the backend under **AddOns → API → Token**.
- Each token has a list of **scopes** (= route names). The token can only call routes whose scope name is in the list.
- Auth header: `Authorization: Bearer <token>`
- Two auth classes:
  - `BearerAuth` — token-based frontend API (default for most routes)
  - `BackendUser` — uses the REDAXO backend session (cookie); for routes that should only be called from the backend (e.g. backend media picker)

### Apache: pass through `Authorization`

Apache strips `Authorization` in some configurations. If the token is correct but the server returns `Authorization failed`, add to `.htaccess`:

```apache
RewriteCond %{HTTP:Authorization} .
RewriteRule ^ - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

## Route table

Authoritative list lives in the addon's `README.md`. Most-used routes:

| Method   | Path                                          | Scope (= route name)              |
|----------|-----------------------------------------------|-----------------------------------|
| GET      | `/api/structure/articles`                     | `structure/articles/list`         |
| POST     | `/api/structure/articles/`                    | `structure/articles/add`          |
| DELETE   | `/api/structure/articles/{id}`                | `structure/articles/delete`       |
| POST     | `/api/structure/articles/{id}/slices`         | `structure/articles/slices/add`   |
| POST     | `/api/structure/categories/`                  | `structure/categories/add`        |
| DELETE   | `/api/structure/categories/{id}`              | `structure/categories/delete`     |
| GET/POST | `/api/templates`                              | `templates/list` / `templates/add`|
| GET/POST | `/api/modules`                                | `modules/list` / `modules/add`    |
| GET/POST | `/api/system/clangs`                          | `system/clangs/list` / `system/clangs/add` |
| GET      | `/api/users`                                  | `users/list`                      |
| GET      | `/api/media`                                  | `media/list`                      |
| GET      | `/api/media/categories`                       | `media/category/list`             |

## Trailing-slash quirks

The addon registers `structure/articles/list` (GET) on `'structure/articles'` (no slash) **and** `structure/articles/add` (POST) on `'structure/articles/'` (with slash). Symfony's `UrlMatcher` is strict, so:

- **GET** must go **without** trailing slash: `/api/structure/articles`
- **POST** must go **with** trailing slash: `/api/structure/articles/`

The wrong variant returns `{"error":"Route with method not found or no access"}` (HTTP 401). That's not an auth problem — the matcher exception is wrapped generically into 401 by `RouteCollection::handle()`. Other endpoints have similar inconsistencies; trust the body schema in the source code over a first-guess URL.

## Two 401 messages — diagnostic split

| Response                                                     | Meaning                                       |
|--------------------------------------------------------------|-----------------------------------------------|
| `{"error":"Authorization failed"}`                           | Token is valid, **scope is missing**          |
| `{"error":"Route with method not found or no access"}`       | Matcher found **no route** (or controller threw) |

For "Route with method not found": check path / method / trailing slash against the source in `src/addons/api/lib/RoutePackage/*.php` first, then suspect the token.

## Diagnostic quick-path

1. **Which auth message?**
   - `Authorization failed` → extend the token's scope list.
   - `Route with method not found or no access` → check path / method / trailing-slash against the source code, then `tail var/log/system.log` for controller PHP errors.
2. **HTTP 500?** → controller threw an unhandled exception. Read the log file.
3. **HTTP 201 but frontend 404?** → yrewrite path cache stale (see below).
4. **Slice add returns 404 with "Template has no module in such ctype"** → in the backend, check the template's module/ctype assignment.

## Known bugs (as of v1.1)

### `structure/articles` list — broken `name` filter

In `Structure::handleArticleList`:

```php
if (null !== $Query['filter']['name']) {       // default is '' not null → branch always runs
    $SqlQueryWhere[':name'] = 'id LIKE :name'; // should be 'name LIKE'
    $SqlParameters[':name'] = '%' . $Query['filter']['id'] . '%'; // 'id' doesn't exist in filter
}
```

Consequences:
- PHP warning `Undefined array key "id"` on every list call
- SQL ends up `id LIKE '%%'` — matches everything but the filter is broken
- Workaround: check `'' === $Query['filter']['name']` instead, then `LIKE %name%` on `name`

### `LIMIT` binding

`rex_sql::factory()->getArray($sql, $params)` binds parameters as strings by default. MySQL doesn't accept `LIMIT 'x', 'y'` in all configurations (especially with `PDO::ATTR_EMULATE_PREPARES=false`). The endpoint then fails with "Route with method not found" (= swallowed exception) even though it's an SQL error.

Workaround: cast `(int)$start` and `(int)$per_page` and inline them into the SQL string instead of using placeholders.

### "Headers already sent"

`RouteCollection::handle()` calls `JSON_PRETTY_PRINT` for the response, then `rex_response::setStatus()` sets headers. Some configurations emit bytes earlier → log warnings (not functionally broken).

### Frontend visibility — yrewrite path cache

A newly-created article/category may return `404` on the frontend even with `status=1`:

- yrewrite caches URL paths in `var/cache/addon/yrewrite/...`
- `rex_article_service::addArticle()` triggers the right EPs, but yrewrite doesn't always regenerate its path cache (especially for new root categories)
- Fix: backend → **AddOns → yrewrite → Allgemein** → "Alle Caches generieren". Or in code: `rex_yrewrite::generatePathFile([])`
- There is currently **no** API endpoint for cache refresh.

### Article metainfo not settable via API

The article `add` schema knows only base fields (`name`, `category_id`, `priority`, `status`, `template_id`). Custom metainfos (`art_*`, `cat_*`) are not accepted. Workaround: set them via a separate backend step or write SQL directly. PUT/PATCH on articles is in the README as planned but not implemented.

### Media upload not active

`POST /api/media` is marked ❌ in the README. Workaround: place media manually (or via migration) in the media pool, then reference filenames via `media1..media10` when creating slices.

## Slice schema

`POST /api/structure/articles/{id}/slices` body:

```json
{
  "module_id": <int, required>,
  "clang_id":  <int, required>,
  "ctype_id":  <int, default 1>,
  "value1":   "...",   "value2":   "...",   ...   "value19":   "...",
  "media1":   "...",   ...   "media10":  "...",
  "medialist1":"...", ...   "medialist10":"...",
  "link1":    "...",   ...   "link10":   "...",
  "linklist1":"...",  ...   "linklist10":"..."
}
```

One slice = one module. Which `value*` / `media*` slot maps to which editor field is defined in `src/modules/<name> [id]/input.php` as `REX_INPUT_VALUE[n]` / `REX_MEDIA[id=n]`.

The addon validates before insert:

- Article exists (`rex_article::get(...)`)
- Language exists (`rex_clang::getAllIds()`)
- Module exists (`rex_module`)
- **Module is allowed in the template's ctype** (`rex_template::hasModule(...)`) — otherwise 404. If this check fails, look at the template's ctype/module assignment in the backend.

## OpenAPI spec

Available in the backend at `?page=api/openapi`. Generated from the route definitions via `OpenAPIConfig`. External consumers (Postman, OpenAPI generator): the JSON spec is only viewable when authenticated via backend login — Swagger UI renders client-side.

## Common pitfalls

- Calling `POST /api/structure/articles` without the trailing slash – 401 "Route with method not found". The matcher cares.
- Adding a scope to the token that doesn't quite match the route name (e.g. `structure/article/add` vs. `structure/articles/add`) – 401 "Authorization failed".
- `Authorization` header missing on Apache – add the `RewriteRule` above.
- Building tooling that posts a slice and expects the article to be reachable on the frontend immediately – regenerate yrewrite path cache after creating articles/categories.
- Reading the swallowed-exception 401 as "wrong token" – check the system log first; lots of 401s are actually 500s in disguise.
