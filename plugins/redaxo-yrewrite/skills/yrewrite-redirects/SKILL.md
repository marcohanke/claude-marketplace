---
name: yrewrite-redirects
description: Configuring redirects and forwards in YRewrite – internal redirects via rex_yrewrite_forward, .htaccess rules, choosing between 301 and 302. Use when the user sets up URL redirects, migrates URLs from a previous site, handles 404s, or fixes broken external links.
---

# YRewrite Redirects & Forwards

YRewrite supports two layers of redirects:

1. **Forwards (database-managed)** – configured in backend (YRewrite → Forwards), stored in `rex_yrewrite_forward`. Best for editor-managed redirects, vanity URLs, post-launch URL fixes.
2. **`.htaccess` / nginx rules** – for high-volume redirects that should never hit PHP (legacy domain → new domain, scheme upgrades).

Use the database layer when editors should manage entries; use the webserver layer for performance-critical or pre-bootstrap rewrites.

## Database forwards

Programmatic creation (e.g. during a migration):

```php
$sql = rex_sql::factory();
$sql->setTable(rex::getTable('yrewrite_forward'));
$sql->setValue('status',     1);
$sql->setValue('clang_start', 1);
$sql->setValue('domain',     'www.example.com');
$sql->setValue('url',        'old/path');           // source: matches request path (no leading /)
$sql->setValue('type',       'extern');             // 'article', 'extern', 'media'
$sql->setValue('target',     'https://new.example.com/landing');
$sql->setValue('redirection_code', 301);
$sql->setValue('description', 'Old marketing landing');
$sql->insert();
```

For `type: article`, set `target` to the article ID and the redirect resolves to its current URL (so it survives URL changes downstream).

For media redirects (e.g. moving a PDF to a new path), use `type: media` and set `target` to the new media filename in the mediapool.

## .htaccess patterns

YRewrite ships an `.htaccess` template in `redaxo/data/addons/yrewrite/.htaccess`. Edit the project's actual `.htaccess` (in the document root), not the template, to add custom rules. Place site-specific rules **above** YRewrite's catch-all so they fire first.

Force HTTPS:

```apacheconf
RewriteEngine On
RewriteCond %{HTTPS} !=on
RewriteRule ^(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
```

Force `www`:

```apacheconf
RewriteCond %{HTTP_HOST} ^example\.com$ [NC]
RewriteRule ^(.*)$ https://www.example.com/$1 [R=301,L]
```

Migrate old URLs in bulk (use a `RewriteMap` or a static block of `RewriteRule`s):

```apacheconf
RewriteRule ^old-product/red-shoe$ /products/red-shoe [R=301,L]
RewriteRule ^old-product/blue-shoe$ /products/blue-shoe [R=301,L]
```

For nginx, the equivalents go in `server` blocks before `try_files $uri $uri/ /index.php?$args;`:

```nginx
if ($host = 'example.com') {
    return 301 https://www.example.com$request_uri;
}
```

## 301 vs 302

| Code | When |
|---|---|
| `301 Moved Permanently` | The old URL is gone forever. Search engines transfer ranking signals. **Default for content moves.** |
| `302 Found` | Temporary – maintenance redirects, A/B testing, geo-routing. Search engines keep the original URL indexed. |
| `307 Temporary Redirect` | Like 302 but preserves the HTTP method on POST. Use for form re-submissions. |
| `308 Permanent Redirect` | Like 301 but preserves the HTTP method. Rare; use 301 unless you need POST preservation. |

In code:

```php
rex_response::sendRedirect($url, 301);
exit; // sendRedirect already exits, but keep this for clarity
```

## Detecting and recording 404s

YRewrite triggers an extension point `YREWRITE_PREPARE` early in the request and sets up the 404 article when no match is found. To log 404s for analysis, hook into `RESPONSE_SHUTDOWN`:

```php
rex_extension::register('RESPONSE_SHUTDOWN', function () {
    if (rex_response::getStatus() !== rex_response::HTTP_NOT_FOUND) {
        return;
    }
    rex_logger::factory()->log('warning', '404 for ' . $_SERVER['REQUEST_URI']);
});
```

Periodically review the log and convert recurring 404s into forward entries.

## Using forwards for vanity URLs

Editors can create forwards with custom slugs that resolve to internal articles:

- Source: `team/alice` (vanity)
- Type: `article`
- Target: 42 (article ID for Alice's profile)
- Code: 301 (or 200/passthrough if the vanity should stay in the URL bar – uses `redirection_code: 0`)

For passthrough (slug stays visible, content from target article), set `redirection_code: 0`. YRewrite re-routes internally without sending a redirect to the browser.

## Common pitfalls

- Using 302 for permanent moves – search engines won't update their index, hurting SEO.
- Creating loops by setting target = source. YRewrite has loop protection but logs the issue and falls through to 404.
- Forgetting `clang_start` on multi-language sites – the forward then matches all languages and may redirect users out of their chosen language.
- Putting forwards in `.htaccess` that editors should manage – they'll change them in the backend and wonder why nothing happens.
- Not testing the redirect with the actual `Host` header. `curl -I -H "Host: www.example.com" https://staging.example.com/old/path` is your friend.
