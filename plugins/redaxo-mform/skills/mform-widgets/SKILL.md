---
name: mform-widgets
description: Advanced MForm widget fields – Custom-Link (addCustomLinkField, addCustomLinkMultipleField, addMFormLinkField), Media picker (addMediaField, addMFormMediaField), Medialist, Imagelist, Linklist, ColorSwatch (addColorSwatchField). Covers all data-* attributes for controlling which link types are enabled (intern/extern/media/mailto/tel), ylink table links, media type restrictions, preview options, and how each widget stores and returns its value. Use when the user adds a link picker, media picker, image gallery, link list, or color swatch to a REDAXO module or YForm form.
---

# MForm Widgets

MForm provides advanced picker widgets beyond standard HTML inputs. All work inside modules and inside the Flex Repeater.

> All examples assume `use FriendsOfRedaxo\MForm;` at the top of the file and `$mform = MForm::factory();` already called.

---

## Custom-Link (`addCustomLinkField`)

A unified link picker supporting internal articles, external URLs, media files, mailto and tel links.

```php
$mform->addCustomLinkField(1, [
    'label'               => 'Link',
    'data-intern'         => 'enable',   // show "internal article" tab
    'data-extern'         => 'enable',   // show "external URL" tab
    'data-media'          => 'enable',   // show "media" tab
    'data-mailto'         => 'enable',   // show "e-mail" tab
    'data-tel'            => 'enable',   // show "telephone" tab
    'data-extern-link-prefix' => 'https://',
    'data-link-category'  => 14,         // open link chooser at this category
    'data-media-category' => 1,          // open media pool at this category
    'data-media-type'     => 'jpg,png,gif', // restrict media types
]);
```

### YForm table link

Link to a row in any YForm-managed table:

```php
$ylink = [['name' => 'Länder', 'table' => 'rex_ycountries', 'column' => 'de_de']];
$mform->addCustomLinkField(1, ['label' => 'Land', 'ylink' => $ylink, 'data-intern' => 'disable']);
```

### Stored value format

The widget stores a single string:
- Internal article: `redaxo://12`
- Media file: `filename.jpg`
- External URL: `https://example.com`
- E-mail: `mailto:user@example.com`
- Telephone: `tel:+4912345`
- YForm row: `rex-tablename://42`

Reading in OUTPUT PHP:
```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$link = 'REX_VALUE[1]';
$url  = MFormOutputHelper::getCustomUrl($link);
$data = MFormOutputHelper::prepareCustomLink(['link' => $link], true);
// $data['customlink_url']    → resolved URL (same as $url)
// $data['customlink_text']   → display text (article name / media title / URL)
// $data['customlink_target'] → ' target="_blank" rel="noopener noreferrer"' for external, else ''

if ($url) {
    echo '<a href="' . rex_escape($url) . '"' . $data['customlink_target'] . '>'
        . rex_escape($data['customlink_text']) . '</a>';
}
```

> See the **mform-output** skill for the full `MFormOutputHelper` API including `createLinkData()`, `normalizeLinkData()`, and `normalizeRepeaterItems()`.

---

## Custom-Link Multiple (`addCustomLinkMultipleField`)

Multiple links as a JSON array in one value slot.

```php
$mform->addCustomLinkMultipleField(1, [
    'label'       => 'Links',
    'btn_add'     => 'Link hinzufügen',
    'data-intern' => 'enable',
    'data-extern' => 'enable',
    'data-media'  => 'enable',
]);
```

Reading in OUTPUT PHP:

Der Wert aus `REX_VALUE` enthält HTML-Entities – deshalb `html_entity_decode()` vor `json_decode()`:

```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$raw   = html_entity_decode('REX_VALUE[1]', ENT_QUOTES | ENT_HTML5, 'UTF-8');
$links = json_decode($raw, true) ?? [];

foreach ($links as $linkStr) {
    $url  = MFormOutputHelper::getCustomUrl($linkStr);
    $data = MFormOutputHelper::prepareCustomLink(['link' => $linkStr], true);
    echo '<a href="' . rex_escape($url) . '"' . $data['customlink_target'] . '>'
        . rex_escape($data['customlink_text']) . '</a>';
}
```

> When reading from a **YForm dataset** (`$dataset->getValue('links')`), skip `html_entity_decode()` – the value is stored without entities. See the **mform-yform** skill.

---

## MForm Link Field (`addMFormLinkField`)

Internal-article-only wrapper around Custom-Link. Same `data-*` attributes apply.

```php
$mform->addMFormLinkField(1, null, 5, ['label' => 'Artikel', 'data-extern' => 'disable', 'data-media' => 'disable']);
```

---

## Media (`addMediaField` / `addMFormMediaField`)

Single file from the media pool.

```php
$mform->addMediaField(1, ['preview' => 1], null, ['label' => 'Bild']);
// Or with category restriction:
$mform->addMediaField(2, ['preview' => 1, 'types' => 'jpg,png,gif,svg,webp'], 3, ['label' => 'Grafik']);

// MForm variant (custom_link-based, stores only filename)
$mform->addMFormMediaField(1, ['preview' => 1], null, ['label' => 'Datei']);
```

Reading in OUTPUT PHP:
```php
$filename = 'REX_VALUE[1]';
$media = rex_media::get($filename);
if ($media) {
    echo '<img src="' . rex_url::media($filename) . '" alt="' . rex_escape($media->getTitle()) . '">';
}
```

---

## Medialist (`addMedialistField`)

Multiple media files; stores a comma-separated filename string.

```php
$mform->addMedialistField(1, ['label' => 'Dateien', 'types' => 'pdf,doc,docx']);
```

Reading in OUTPUT PHP:
```php
$filenames = array_filter(explode(',', 'REX_VALUE[1]'));
foreach ($filenames as $filename) {
    $media = rex_media::get($filename);
    if ($media) {
        echo '<a href="' . rex_url::media($filename) . '">' . rex_escape($media->getTitle()) . '</a>';
    }
}
```

---

## Imagelist (`addImagelistField`)

Image gallery with view-toggle (grid / list / gallery). Stores a comma-separated filename string.

```php
$mform->addImagelistField(1, ['label' => 'Bildergalerie', 'types' => 'jpg,jpeg,png,webp,gif']);
```

Reading in OUTPUT PHP:
```php
$filenames = array_filter(explode(',', 'REX_VALUE[1]'));
foreach ($filenames as $filename) {
    echo '<img src="' . rex_url::media($filename) . '" alt="">';
}
```

---

## Linklist (`addLinklistField`)

Multiple internal article links; stores a comma-separated list of article IDs.

```php
$mform->addLinklistField(1, null, 5, ['label' => 'Verwandte Artikel']);
```

Reading in OUTPUT PHP:
```php
$ids = array_filter(explode(',', 'REX_VALUE[1]'));
foreach ($ids as $id) {
    $art = rex_article::get((int) $id);
    if ($art) {
        echo '<a href="' . rex_getUrl($art->getId()) . '">' . rex_escape($art->getName()) . '</a>';
    }
}
```

---

## ColorSwatch (`addColorSwatchField`)

Text input + visual color/class picker. Stores either a hex color (`#2f77bc`) or a CSS class name (`.bg-primary`).

```php
$mform->addColorSwatchField(1, [
    '#ffffff' => 'Weiß',
    '#000000' => 'Schwarz',
    '#2f77bc' => 'Primär',
    '.bg-warning' => ['label' => 'Warnung', 'preview' => '#f0ad4e'],  // CSS class with preview color
], ['label' => 'Hintergrundfarbe'], '#ffffff');
```

Reading in OUTPUT PHP:
```php
$color = 'REX_VALUE[1]';
if (str_starts_with($color, '.')) {
    // CSS class
    echo '<div class="' . rex_escape(ltrim($color, '.')) . '">';
} else {
    // Hex value
    echo '<div style="background-color:' . rex_escape($color) . '">';
}
```

---

## Common pitfalls

- **`data-intern` / `data-extern` / `data-media` default to `enable`** unless explicitly set to `disable`. All tabs are shown unless you restrict them.
- **Medialist and Imagelist store comma-separated filenames**, not JSON. Use `explode(',', $val)`, not `json_decode()`.
- **Linklist stores comma-separated article IDs** as plain integers, not `redaxo://` format.
- **`addMediaField()` (REDAXO native) vs `addMFormMediaField()` (MForm custom)**: both store a filename string. `addMFormMediaField` uses the Custom-Link widget internally and has slightly different UI.
- **ColorSwatch CSS-class values start with `.`** – remember to strip the leading dot when using as an HTML class attribute: `ltrim($val, '.')`.
- **`ylink` on Custom-Link requires YForm** – ensure YForm is available before adding such a field.
