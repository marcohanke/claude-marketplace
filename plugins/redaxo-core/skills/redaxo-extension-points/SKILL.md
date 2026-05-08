---
name: redaxo-extension-points
description: REDAXO Extension Points – the hook system for modifying core/addon behavior without patching files. Use when the user mentions rex_extension::register, hooks into events like ART_PRE_VIEW / OUTPUT_FILTER / PACKAGES_INCLUDED, or wants to extend backend pages and content.
---

# Extension Points (EPs)

Extension Points are REDAXO's hook system. Instead of patching core or another addon, you register a callback that fires at a defined moment.

Registration goes in your addon's `boot.php`:

```php
<?php
// boot.php
rex_extension::register('OUTPUT_FILTER', function (rex_extension_point $ep) {
    $content = $ep->getSubject();
    // modify $content...
    return $content;
});
```

The return value replaces the subject for downstream listeners. Returning `null` (or no return) keeps the subject unchanged.

## Signature & priority

```php
rex_extension::register(
    string $extensionPoint,
    callable $callback,
    int $level = rex_extension::NORMAL,  // EARLY, NORMAL, LATE
    array $params = []                    // extra data the EP receives
);
```

Lower level values run first. Use `EARLY` to run before the default behavior, `LATE` for cleanup or final adjustments.

## Common extension points

### Frontend rendering

| EP | When | Subject | Use for |
|---|---|---|---|
| `OUTPUT_FILTER` | After full HTML is built | full page HTML | minify, inject scripts, replace tokens |
| `OUTPUT_FILTER_CACHE` | Before HTML is cached | full page HTML | sanitize before persisting cache |
| `ART_PRE_VIEW` | Before article renders | rendered article HTML | inject blocks, wrap content |
| `ART_INIT` | Article object created | `rex_article_content` | conditionally swap template |

### System lifecycle

| EP | When | Use for |
|---|---|---|
| `PACKAGES_INCLUDED` | After all addons booted | cross-addon setup that needs siblings loaded |
| `READY` | After core+addons are ready, before request dispatch | last-chance configuration |
| `RESPONSE_SHUTDOWN` | After response was sent | async cleanup, stats |

### Backend / content

| EP | When | Use for |
|---|---|---|
| `STRUCTURE_CONTENT_HEADER` | Backend article edit – above slices | warning banners, info |
| `SLICE_SHOW` | Each slice in edit mode | extra controls per slice |
| `MEDIA_IS_IN_USE` | Before media deletion | prevent deletion if you reference it elsewhere |
| `CLANG_DELETED` | A language was removed | clean up your `clang`-keyed data |
| `ART_DELETED` / `CAT_DELETED` | An article/category is removed | clean up references |

The full list is documented at <https://redaxo.org/doku/main/extension-points>.

## Reading params and subject

Inside the callback, the `rex_extension_point` instance carries:

- `$ep->getSubject()` – the value passed in (often what you'll modify)
- `$ep->getParam('key', $default)` – extra context (article ID, user, etc.)
- `$ep->setParam('key', $value)` – pass info to later listeners
- `$ep->getName()` – the EP name (useful for shared callbacks)

```php
rex_extension::register('ART_PRE_VIEW', function (rex_extension_point $ep) {
    $articleId = $ep->getParam('article_id');
    $clang     = $ep->getParam('clang');
    $html      = $ep->getSubject();

    if ($articleId === rex_article::getNotfoundArticleId()) {
        return $html; // don't touch 404s
    }

    return preg_replace('/<\/body>/', '<!-- rendered -->\n</body>', $html);
});
```

## Backend-only extension

Always guard backend-only logic:

```php
rex_extension::register('PACKAGES_INCLUDED', function () {
    if (!rex::isBackend()) {
        return;
    }
    if (!rex::getUser() || !rex::getUser()->isAdmin()) {
        return;
    }
    // …admin-only setup
});
```

## Defining your own EP

Lets other addons hook into your code:

```php
$result = rex_extension::registerPoint(new rex_extension_point(
    'MY_ADDON_ITEM_SAVED',
    $itemHtml,                        // subject
    ['item_id' => $id, 'user' => rex::getUser()] // params
));
```

Subscribers can transform `$result` and your code continues with their version.

## Common pitfalls

- Registering inside a function that isn't called (e.g. inside an `if (rex::isFrontend())` that fires only on frontend). Register unconditionally in `boot.php`, then guard inside the callback.
- Forgetting to `return $ep->getSubject()` (or your modified version) – downstream listeners then see `null`.
- Using `OUTPUT_FILTER` for things that should be in the template – it runs on every uncached request and adds latency.
- Heavy work in `PACKAGES_INCLUDED` slows down every request. Cache results or move into a cronjob.
