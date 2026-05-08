---
name: search-it-search
description: SearchIt full-text search class – executing searches, filtering by categories/articles/columns, result array structure, hit types (article/db_column/file/url), highlighting, pagination, similarity search. Use when the user works with FriendsOfRedaxo\SearchIt\SearchIt, search_it, $search->search(), $result['hits'], search result output, or asks how to search in REDAXO.
---

# SearchIt – Executing Searches

The main class is `FriendsOfRedaxo\SearchIt\SearchIt` (namespace since v7; legacy alias `search_it` still works but is deprecated).

## Constructor

```php
use FriendsOfRedaxo\SearchIt\SearchIt;

$search = new SearchIt(
    int|false $clang = false,       // language ID or false for all languages
    bool $loadSettings = true,       // load addon backend settings
    bool $useStopwords = true        // apply German stopword list
);
```

## Basic search

```php
$searchTerm = rex_request('search', 'string', '');
if ($searchTerm !== '') {
    $search = new SearchIt();
    $result = $search->search($searchTerm);
}
```

## Filtering results

```php
// Only specific articles
$search->searchInArticles([1, 5, 12]);

// Only articles in specific categories
$search->searchInCategories([3, 7]);

// Category tree (category + all subcategories)
$search->searchInCategoryTree(3);

// Only specific media pool categories
$search->searchInFileCategories([2, 4]);

// Only specific DB columns
$search->searchInDbColumn('rex_article', 'name');

// Custom WHERE clause on the index table
$search->setWhere("texttype = 'article'");
```

### YRewrite multi-domain: restrict to one domain

```php
$search->searchInCategoryTree(
    rex_yrewrite::getDomainByName('example.com')->getMountId()
);
```

## Search behaviour

```php
// Logical mode: 'and' (all terms must match) or 'or' (any term)
$search->setLogicalMode('and');

// Text mode: 'plain' (plaintext index), 'unmodified' (HTML), 'both'
$search->setTextMode('plain');

// MySQL search mode: 'like' (LIKE %term%) or 'match' (FULLTEXT MATCH)
$search->setSearchMode('like');

// Weight specific words higher
$search->addWhitelist(['important' => 5, 'redaxo' => 3]);

// Blacklist words
$search->setBlacklist(['stopword1', 'stopword2']);
```

## Highlighting

```php
// Surround tags (default: <mark>…</mark>)
$search->setSurroundTags('<strong>', '</strong>');

// Highlight type
$search->setHighlightType('surroundtext');
// Options: 'sentence', 'paragraph', 'surroundtext', 'surroundtextsingle', 'teaser', 'array'

// Max chars for highlighted text
$search->setMaxHighlightedTextChars(200);
```

## Pagination

```php
$page = rex_request('page', 'int', 1);
$perPage = 10;
$search->setLimit(($page - 1) * $perPage, $perPage);
$result = $search->search($searchTerm);

$totalPages = (int) ceil($result['count'] / $perPage);
```

## Sorting

```php
$search->setOrder(['clang' => 'ASC']);
```

## Result array structure

```php
$result = $search->search('term');

$result['count']              // int – total hits (ignoring LIMIT)
$result['hits']               // array – hit objects
$result['keywords']           // array – parsed search terms with weights
$result['searchterm']         // string – original search string
$result['time']               // float – search duration in seconds
$result['simwords']           // array – similar words (if enabled)
$result['simwordsnewsearch']  // string – suggested search with similar words
$result['blacklisted']        // array|false – blacklisted words found in query
$result['hash']               // string – cache hash
```

### Single hit (`$result['hits'][$i]`)

| Key | Type | Description |
|-----|------|-------------|
| `fid` | int/string | Foreign ID (article ID, URL hash, DB primary key) |
| `type` | string | `'article'`, `'url'`, `'db_column'`, `'file'` |
| `table` | string | Source table (e.g. `rex_article`) |
| `column` | string/null | Source column (only `db_column` type) |
| `clang` | int/null | Language ID |
| `plaintext` | string | Indexed plaintext |
| `highlightedtext` | string | Text with highlighted search terms |
| `teaser` | string | Short teaser text |
| `article_teaser` | string | Teaser from article content |
| `values` | array/null | Additional indexed column values |
| `filename` | string/null | Filename (only `file` type) |
| `fileext` | string/null | File extension (only `file` type) |

### Distinguishing hit types

```php
foreach ($result['hits'] as $hit) {
    switch ($hit['type']) {
        case 'article':
            $url = rex_getUrl($hit['fid'], $hit['clang']);
            $name = rex_article::get($hit['fid'], $hit['clang'])?->getName();
            break;

        case 'db_column':
            if ($hit['table'] === rex::getTable('article')) {
                $url = rex_getUrl($hit['fid'], $hit['clang']);
            }
            break;

        case 'file':
            $url = rex_url::media($hit['filename']);
            break;

        case 'url':
            // URL addon hit – reconstruct URL via profile
            break;
    }
}
```

## Similarity search ("Did you mean?")

Requires activation in backend settings. Builds a keyword index from successful searches over time.

```php
$result = $search->search($searchTerm);

if ($result['count'] === 0 && !empty($result['simwordsnewsearch'])) {
    echo 'Did you mean: <a href="?search=' . urlencode($result['simwordsnewsearch']) . '">'
        . rex_escape($result['simwordsnewsearch']) . '</a>';
}
```

## Namespace structure (v7+)

```
FriendsOfRedaxo\SearchIt\
├── SearchIt                     – main class (search + indexing)
├── Api\Autocomplete             – autocomplete API endpoint
├── Cache\SearchCache            – search result cache
├── Console\ReindexCommand       – console: search_it:reindex
├── Console\ClearCacheCommand    – console: search_it:clearCache
├── Cronjob\Reindex              – cronjob: reindex
├── Cronjob\ClearCache           – cronjob: clear cache
├── EventHandler                 – REDAXO extension point handler
├── Helper\ArticleHelper         – article/category lists
├── Helper\ColognePhonetic       – Cologne phonetic (similarity)
├── Helper\FileHelper            – file/directory scanning
├── Helper\FormBuilder           – backend settings forms
├── Helper\SocketHelper          – HTTP socket for indexing
├── Helper\UrlAddon              – URL addon integration
├── Index\KeywordStore           – keyword storage
├── Pdf\PdfConverter             – PDF-to-text conversion
├── Plaintext\PlaintextConverter – HTML-to-plaintext
├── Search\Highlighter           – frontend keyword highlighter
├── Stats\Statistics             – search statistics
```

## Common pitfalls

- Forgetting `rex_request('search', 'string', '')` – never use `$_GET['search']` directly.
- Calling `$search->search()` without checking for empty search term – returns all indexed content, slow on large sites.
- Mixing up `searchInCategories()` (direct children only) and `searchInCategoryTree()` (recursive) – use the tree variant for section-scoped search.
- Using `setSearchMode('match')` without a FULLTEXT index on the search_it tables – falls back silently to slow queries.
- Not regenerating the index after adding new `searchInDbColumn()` sources in the backend.
