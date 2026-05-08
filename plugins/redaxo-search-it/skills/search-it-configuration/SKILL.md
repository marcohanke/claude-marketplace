---
name: search-it-configuration
description: Search It backend configuration – search modes (LIKE/MATCH, AND/OR), text modes, highlight settings, additional index sources (DB columns, media, files), blacklist, plaintext conversion, similarity search (soundex/metaphone/Cologne phonetic), autocomplete/suggest setup, statistics. Use when the user configures search_it settings, adds index sources, or asks about search modes and options.
---

# Search It – Configuration

All settings are managed in the REDAXO backend under Search It > Settings. They are stored in `rex_config` with addon key `search_it`.

## Search mode settings

### Logical mode

- **AND** – all search terms must appear (default, stricter)
- **OR** – any search term matches (broader results)

### Text mode

- **plain** – search in plaintext index (recommended, ignores HTML)
- **unmodified** – search in original HTML content
- **both** – search in both (slower, more results)

### Search mode (MySQL)

- **LIKE** – `LIKE %term%` queries, works always, slower on large indexes
- **MATCH** – MySQL FULLTEXT `MATCH ... AGAINST`, faster but requires FULLTEXT index and minimum word length (default 4 chars in MySQL)

## Result settings

- **Highlight type** – how matched text is presented:
  - `sentence` – full sentence containing the match
  - `paragraph` – full paragraph
  - `surroundtext` – surrounding characters around the match
  - `surroundtextsingle` – surrounding characters, only first match
  - `teaser` – fixed-length teaser from start of content
  - `array` – returns array of matches (for custom rendering)
- **Surround tags** – HTML tags wrapping matches (default: `<mark>`, `</mark>`)
- **Max highlighted chars** – character limit for highlighted text
- **Max results** – hard limit on total results

## Additional index sources

Under Settings > Additional sources, configure:

### Database columns

Add any `table.column` pair. Common examples:

| Table | Column | Purpose |
|-------|--------|---------|
| `rex_article` | `name` | Article names in search results |
| `rex_media` | `title` | Media titles |
| `rex_media` | `med_description` | Media descriptions |
| Custom YForm table | `name`, `description` | Custom data records |

After adding columns, trigger a full reindex.

### File indexing

- Enable/disable file indexing
- Allowed file extensions (e.g. `pdf,doc,docx,txt`)
- Custom directories to scan (beyond media pool)

### URL addon

If the URL addon is installed, enable URL indexing to include URL addon profile URLs in the search index.

## Blacklist

### Blacklisted words

Words that are stripped from search queries. Useful for very common terms that produce too many results.

### Blacklisted articles

Specific article IDs excluded from indexing entirely.

### Blacklisted categories

All articles in these categories (and optionally subcategories) are excluded from the index.

## Similarity search

Enables "Did you mean?" suggestions when a search returns no results. Three phonetic algorithms available:

| Mode | Algorithm | Best for |
|------|-----------|----------|
| 1 | Soundex | English words |
| 2 | Metaphone | English words (better than Soundex) |
| 4 | Cologne Phonetic | German words |
| 7 | All combined | Mixed content |

Enable in backend and rebuild the keyword index. The keyword index grows over time as users search – it learns from real queries.

## Autocomplete / Suggest

1. Enable in Settings > Suggest
2. Configure minimum characters, maximum suggestions, search delay
3. The addon generates a JS snippet – embed it in your template before `</body>`
4. Search form must have `class="search_it-form"` and input `name="search"`

Generated JS code is available via:

```php
rex_addon::get('search_it')->getProperty('suggest_js', '');
```

Alternatively, include the addon's CSS and JS assets manually:

```html
<link rel="stylesheet" href="<?= rex_url::addonAssets('search_it', 'css/suggest.css') ?>">
<script src="<?= rex_url::addonAssets('search_it', 'js/suggest.js') ?>"></script>
```

## Plaintext settings

Control how HTML content is converted to plaintext for indexing:

- **CSS selectors to exclude** – elements matching these selectors are removed before indexing (e.g. `nav, footer, .search-exclude`)
- **Regex patterns** – custom regex replacements applied during conversion
- **Textile parsing** – enable if content uses Textile markup

## Statistics

When enabled, Search It tracks:
- Search terms used
- Frequency of each term
- Results count per term

View in backend under Search It > Statistics. Useful for understanding what visitors search for and improving content.

## Indexing settings

- **Host / Port** – for socket-based article fetching during indexing (usually `localhost:80`)
- **SSL verification** – disable for self-signed certificates in development
- **Logging** – enable to log indexing errors to `redaxo/data/addons/search_it/`

## Common pitfalls

- Setting search mode to MATCH but MySQL's `ft_min_word_len` is 4 – three-letter words are silently ignored. Check with `SHOW VARIABLES LIKE 'ft_min_word_len'`.
- Enabling Cologne Phonetic for an English-only site – produces poor suggestions. Use Metaphone instead.
- Configuring CSS selectors to exclude content but not reindexing – old content remains in the index.
- Enabling autocomplete but forgetting the `search_it-form` class on the form – JS doesn't bind.
- Blacklisting common words that are actually important search terms for the specific site.
