---
name: search-it-extension-points
description: Search It extension points – SEARCH_IT_INDEX_ARTICLE, SEARCH_IT_PLAINTEXT, SEARCH_IT_SEARCH_EXECUTED. Use when the user wants to hook into search indexing, customise plaintext conversion, exclude articles from the index, track search analytics, or modify search behaviour via rex_extension.
---

# Search It – Extension Points

Search It provides three extension points for customising indexing and search behaviour. Register them in your project addon's `boot.php`.

## SEARCH_IT_INDEX_ARTICLE

Fires **before** an article is indexed. Return `false` to exclude the article from the index.

**Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `article` | `rex_article` | The article about to be indexed |

### Exclude articles by meta field

```php
rex_extension::register('SEARCH_IT_INDEX_ARTICLE', function(rex_extension_point $ep) {
    $article = $ep->getParam('article');

    // Skip articles with noindex meta field
    if ($article->getValue('art_noindex') == 1) {
        return false;
    }
});
```

### Exclude offline categories

```php
rex_extension::register('SEARCH_IT_INDEX_ARTICLE', function(rex_extension_point $ep) {
    $article = $ep->getParam('article');
    $category = $article->getCategory();

    if ($category && !$category->isOnline()) {
        return false;
    }
});
```

### Exclude specific templates

```php
rex_extension::register('SEARCH_IT_INDEX_ARTICLE', function(rex_extension_point $ep) {
    $article = $ep->getParam('article');
    $excludedTemplates = [5, 8]; // template IDs not to index

    if (in_array($article->getTemplateId(), $excludedTemplates)) {
        return false;
    }
});
```

## SEARCH_IT_PLAINTEXT

Fires during plaintext conversion, **after** the article HTML has been fetched but **before** the standard plaintext processing. Allows you to modify or replace the text that gets indexed.

**Subject:** `string` – the raw HTML/text content

### Remove specific elements before indexing

```php
rex_extension::register('SEARCH_IT_PLAINTEXT', function(rex_extension_point $ep) {
    $text = $ep->getSubject();

    // Remove elements that should not appear in search results
    $text = preg_replace('/<nav[^>]*>.*?<\/nav>/si', '', $text);
    $text = preg_replace('/<div class="no-search">.*?<\/div>/si', '', $text);

    return $text;
});
```

### Return array to control processing pipeline

```php
rex_extension::register('SEARCH_IT_PLAINTEXT', function(rex_extension_point $ep) {
    $text = $ep->getSubject();

    // Custom processing
    $text = strip_tags($text, '<p><br>');
    $text = html_entity_decode($text, ENT_QUOTES, 'UTF-8');

    return [
        'text' => $text,
        'process' => false  // false = skip built-in plaintext conversion
                            // true  = run built-in conversion after this hook
    ];
});
```

### Add extra content to the index

```php
rex_extension::register('SEARCH_IT_PLAINTEXT', function(rex_extension_point $ep) {
    $text = $ep->getSubject();
    $params = $ep->getParams();

    // Append YForm data related to this article
    $articleId = $params['article_id'] ?? null;
    if ($articleId) {
        $extra = rex_sql::factory()
            ->setQuery('SELECT description FROM ' . rex::getTable('my_data') . ' WHERE article_id = ?', [$articleId]);

        if ($extra->getRows() > 0) {
            $text .= ' ' . $extra->getValue('description');
        }
    }

    return $text;
});
```

## SEARCH_IT_SEARCH_EXECUTED

Fires **after** every search query is executed. Useful for logging, analytics, or modifying results.

**Subject:** `array` – the full result array (same structure as `$search->search()` returns)

### Log searches to a custom table

```php
rex_extension::register('SEARCH_IT_SEARCH_EXECUTED', function(rex_extension_point $ep) {
    $result = $ep->getSubject();

    rex_sql::factory()
        ->setTable(rex::getTable('search_log'))
        ->setValue('searchterm', $result['searchterm'])
        ->setValue('results_count', $result['count'])
        ->setValue('search_time', $result['time'])
        ->setValue('createdate', date('Y-m-d H:i:s'))
        ->insert();
});
```

### Track zero-result searches

```php
rex_extension::register('SEARCH_IT_SEARCH_EXECUTED', function(rex_extension_point $ep) {
    $result = $ep->getSubject();

    if ($result['count'] === 0) {
        rex_logger::factory()->log('search_it', 'Zero results for: ' . $result['searchterm']);
    }
});
```

### Modify results (filter, reorder)

```php
rex_extension::register('SEARCH_IT_SEARCH_EXECUTED', function(rex_extension_point $ep) {
    $result = $ep->getSubject();

    // Remove file hits from results
    $result['hits'] = array_filter($result['hits'], function($hit) {
        return $hit['type'] !== 'file';
    });

    $result['hits'] = array_values($result['hits']); // reindex
    return $result;
});
```

## Registration location

Always register these extension points in your **project addon's `boot.php`** (or a custom addon's `boot.php`), never in a module or template:

```php
// redaxo/src/addons/project/boot.php

rex_extension::register('SEARCH_IT_INDEX_ARTICLE', function(rex_extension_point $ep) {
    // ...
});
```

## Common pitfalls

- Registering extension points in a module output – they run only when that article is displayed, not during indexing. Always use `boot.php`.
- Returning `null` instead of `false` in `SEARCH_IT_INDEX_ARTICLE` – `null` does not exclude the article. You must explicitly `return false`.
- Using heavy database queries in `SEARCH_IT_PLAINTEXT` – this runs for every article during reindex. Keep it lightweight or cache results.
- Modifying `$result['hits']` in `SEARCH_IT_SEARCH_EXECUTED` without updating `$result['count']` – pagination breaks if count no longer matches actual hits.
- Forgetting `return` in `SEARCH_IT_SEARCH_EXECUTED` when modifying results – without returning the modified array, changes are lost.
