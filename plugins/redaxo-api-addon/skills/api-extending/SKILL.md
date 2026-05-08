---
name: api-extending
description: Adding custom REST routes to the FriendsOfRedaxo/api addon from your own addon – building a RoutePackage subclass, registering routes, scope management, and the request-context caveats (auth runs in frontend context, isBackend() returns false). Use when the user adds endpoints to the api addon, exposes their own data over the same auth/scope mechanism, or runs into "EP fires only in backend but I need it from the API" problems.
---

# Extending the `api` Addon

You can add your own routes to the `api` addon from any custom addon. They reuse the same Bearer token, the same scope mechanism, and the same OpenAPI generator as the built-in routes.

## RoutePackage skeleton

```php
<?php
namespace MyVendor\MyAddon\RoutePackage;

use FriendsOfRedaxo\Api\Auth\BearerAuth;
use FriendsOfRedaxo\Api\RouteCollection;
use FriendsOfRedaxo\Api\RoutePackage;
use Symfony\Component\Routing\Route;

class MyResource extends RoutePackage
{
    public function loadRoutes(): void
    {
        RouteCollection::registerRoute(
            'myaddon/things/list',                          // becomes the token scope name
            new Route(
                'myaddon/things',                            // → /api/myaddon/things
                ['_controller' => self::class . '::handleList', 'query' => [/* schema */]],
                [], [], '', [], ['GET'],
            ),
            'List things',                                   // OpenAPI summary
            null,                                            // OpenAPI description
            new BearerAuth(),
        );
    }

    public static function handleList($Parameter): \Symfony\Component\HttpFoundation\Response
    {
        // ... build response data ...
        return new \Symfony\Component\HttpFoundation\Response(json_encode([
            'data' => [/* ... */],
        ]));
    }
}
```

## Register in the addon's `boot.php`

```php
\FriendsOfRedaxo\Api\RouteCollection::registerRoutePackage(
    new \MyVendor\MyAddon\RoutePackage\MyResource()
);
```

After registration, the new scope (`myaddon/things/list`) appears in the backend token editor. Add it to the relevant token, otherwise calls return `Authorization failed`.

## Trailing-slash decision

Stay consistent: pick either trailing-slash or no-trailing-slash per resource and keep all methods on that resource using the same convention. The built-in addon mixed it up; don't replicate that mistake.

```php
// Recommended: no trailing slash for all methods on /things
'myaddon/things'         // GET   → list
'myaddon/things'         // POST  → create
'myaddon/things/{id}'    // GET   → show
'myaddon/things/{id}'    // PATCH → update
'myaddon/things/{id}'    // DELETE → remove
```

## Authentication options

| Class | Use for |
|---|---|
| `BearerAuth` | Standard token-based API |
| `BackendUser` | Routes that should only be callable when logged into the backend (cookie session) |
| `null` | Unauthenticated public endpoint — only do this for actually public data |

Prefer `BearerAuth` even for "internal" endpoints. The token + scope mechanism beats IP allowlists for auditability.

## Request context caveats

`RouteCollection::handle()` runs in the **frontend user context**. Two consequences:

- Extension points registered only in `rex::isBackend()` branches **don't fire**. If your addon has hooks registered like `if (rex::isBackend()) { rex_extension::register(...) }`, those hooks are silent for API calls. Either register them unconditionally and guard inside the callback, or trigger them explicitly inside the API route.
- `rex_request::isBackend()` returns `false` in API calls.

When porting backend logic to an API endpoint, audit every `rex::isBackend()` and `rex::getUser()` check in the call path — backend admin permissions don't apply.

## Returning the right response shape

For consistency with the built-in routes:

- 200 with a body for successful reads
- 201 with the created resource for successful creates
- 204 (empty) for successful deletes
- 4xx with `{"error": "..."}` for client errors
- 5xx with `{"error": "..."}` only after logging the underlying exception

Use `\Symfony\Component\HttpFoundation\Response` directly. JSON encoding is your responsibility.

```php
return new \Symfony\Component\HttpFoundation\Response(
    json_encode(['data' => $items]),
    200,
    ['Content-Type' => 'application/json']
);
```

## OpenAPI integration

The addon generates an OpenAPI spec at `?page=api/openapi` from registered routes. The third and fourth arguments to `registerRoute()` (summary + description) feed that generator. Spend a minute writing a meaningful summary — Swagger UI uses it.

Schema for query parameters is passed via the `query` key in `_controller`:

```php
['_controller' => self::class . '::handleList', 'query' => [
    'filter' => ['type' => 'object', 'properties' => [
        'name' => ['type' => 'string'],
    ]],
    'page'     => ['type' => 'integer', 'default' => 1],
    'per_page' => ['type' => 'integer', 'default' => 20],
]]
```

The OpenAPI generator mirrors this into the spec.

## Common pitfalls

- Forgetting to register the RoutePackage in `boot.php` – the route never appears, calls return 404.
- Forgetting to add the new scope to the token in the backend – calls return `Authorization failed`.
- Throwing exceptions in the controller without logging – `RouteCollection::handle()` swallows them into a generic 401 / 500. Always `try { ... } catch (Throwable $e) { rex_logger::logException($e); throw; }` (or return a structured error).
- Building a controller that calls into code paths gated on `rex::isBackend()` – they silently skip in API context.
- Using closures in `_controller` instead of `Class::method` references – Symfony's `UrlMatcher` doesn't serialize closures cleanly across the routing layer.
- Accepting JSON bodies via `$_POST` – read `php://input` instead and `json_decode()` it.
