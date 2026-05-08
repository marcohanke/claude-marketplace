---
name: structure-content
description: Slice management in REDAXO – the content blocks that make up an article. Use when the user creates/edits/deletes slices programmatically, lists slices of an article, copies content between articles, or syncs content between languages.
---

# Slices & Article Content

Article content in REDAXO consists of "slices" – ordered instances of modules with their `REX_VALUE` / `REX_MEDIA` / `REX_LINK` data filled in. Slices are stored in `rex_article_slice`.

For most use cases, you don't manipulate slices directly – the backend UI handles it. The API matters when:

- Importing content from external sources
- Cloning or templating content across articles
- Building custom workflows (e.g. duplicate to all languages)

## Reading a slice

`rex_article_slice` provides one slice at a time:

```php
$slice = rex_article_slice::getSliceById($sliceId);
echo $slice->getModuleId();
echo $slice->getValue(1);   // REX_VALUE[1]
echo $slice->getMedia(1);   // REX_MEDIA[1] – returns filename string
echo $slice->getMedialist(1); // comma-separated filenames
echo $slice->getLink(1);    // REX_LINK[1] – article ID as string
echo $slice->getLinklist(1);
```

## Listing slices of an article

```php
$slices = rex_article_slice::getSlicesForArticle($articleId, $clangId);
foreach ($slices as $slice) {
    echo "Slice #{$slice->getId()} module #{$slice->getModuleId()}\n";
}
```

## Adding a slice programmatically

Use `rex_content_service::addSlice()` so the priority shift, cache invalidation, and `SLICE_ADDED` extension point all happen.

```php
$articleId = 42;
$clangId   = rex_clang::getCurrentId();
$ctype     = 1;  // content area (templates may define multiple)
$moduleId  = 7;

$data = [
    'value1' => 'Hello world',
    'value2' => 'Some longer text',
    'media1' => 'hero.jpg',
    'link1'  => '5',
];

$sliceId = rex_content_service::addSlice($articleId, $clangId, $ctype, $moduleId, $data);
```

The `$data` keys are `value1`–`value20`, `media1`–`media10`, `medialist1`–`medialist10`, `link1`–`link10`, `linklist1`–`linklist10`.

## Editing / deleting

```php
rex_content_service::editSlice($sliceId, [
    'value1' => 'Updated headline',
]);

rex_content_service::deleteSlice($sliceId);
```

## Moving (re-ordering)

```php
rex_content_service::moveSlice($sliceId, 'up');   // or 'down'
```

For exact priority, edit `priority` directly:

```php
rex_content_service::editSlice($sliceId, ['priority' => 3]);
```

## Cloning content between languages

Common workflow: "create a German article, then duplicate its slices into the English variant for translation".

```php
function copySlicesAcrossClang(int $articleId, int $fromClang, int $toClang): void
{
    $slices = rex_article_slice::getSlicesForArticle($articleId, $fromClang);
    foreach ($slices as $slice) {
        $data = [
            'priority' => $slice->getPriority(),
        ];
        for ($i = 1; $i <= 20; $i++) {
            $data['value' . $i] = $slice->getValue($i);
        }
        for ($i = 1; $i <= 10; $i++) {
            $data['media' . $i]     = $slice->getMedia($i);
            $data['medialist' . $i] = $slice->getMedialist($i);
            $data['link' . $i]      = $slice->getLink($i);
            $data['linklist' . $i]  = $slice->getLinklist($i);
        }
        rex_content_service::addSlice(
            $articleId,
            $toClang,
            $slice->getCtype(),
            $slice->getModuleId(),
            $data
        );
    }
}
```

## Copy a whole article

`rex_article_service::copyArticle()` clones the article record only. To clone with content, follow up with `rex_content_service::copyContent()`:

```php
$newId = rex_article_service::copyArticle($sourceId, $targetCategoryId);
rex_content_service::copyContent($sourceId, $newId, $clangId, $clangId);
```

## Rendering article content from PHP

Inside templates, use `REX_ARTICLE[]`. Outside (e.g. cron job, CLI), use `rex_article_content`:

```php
$content = new rex_article_content($articleId, $clangId);
$content->setTemplateId(0); // 0 = no template wrapper
$html = $content->getArticle();
```

This is how RSS-feed addons, search indexers, and PDF exporters extract article HTML.

## Common pitfalls

- Inserting slices via raw SQL – misses `priority` shifting and cache flush, leading to broken ordering and stale cached pages.
- Forgetting `ctype` – defaults to 1, which works only if your template defines a single content area.
- Hardcoding module IDs in code – they differ between staging and production. Look them up by name on first run and cache:

```php
$sql = rex_sql::factory();
$sql->setQuery('SELECT id FROM ' . rex::getTable('module') . ' WHERE name = ?', ['Hero banner']);
$moduleId = $sql->getRows() ? (int) $sql->getValue('id') : null;
```

- Calling `getValue(0)` instead of `getValue(1)` – placeholders are 1-indexed.
