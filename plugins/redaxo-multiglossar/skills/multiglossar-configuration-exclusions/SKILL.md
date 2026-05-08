---
name: multiglossar-configuration-exclusions
description: MultiGlossar configuration and exclusion rules - glossar_starttag/glossar_endtag scope selection, glossar_ignoretags with tag/class locks, article/template/metainfo exclusion settings, cache and turbocache options, and yrewrite/domain-specific glossary article mapping (article_<domainId>). Use when the user configures where replacements happen or why certain pages/articles must be skipped.
---

# MultiGlossar Configuration and Exclusions

MultiGlossar behavior is mostly controlled through addon settings. Correct configuration is essential to avoid over-replacement or performance issues.

## Scope configuration (start/end tags)

Replacement is applied only within one configured document segment:

- `glossar_starttag` - regex/string for start marker (default around `<body.*?>`)
- `glossar_endtag` - regex/string for end marker (default `</body>`)

Projects can use explicit HTML comments in templates (for example `<!--glossar_start-->` and `<!--glossar_stop-->`) and configure these markers in settings.

Only one unique scope is expected. Ambiguous patterns can lead to missing replacements.

## Ignored tags and classes

MultiGlossar has built-in ignored elements (for example `a`, `h1`-`h6`, `figcaption`, `script`, `style`, `svg`, `dfn`, `exclude`).

Additional excludes can be set via `glossar_ignoretags`:

- tag names, e.g. `nav,aside,ul`
- class selectors with dot prefix, e.g. `.no-glossary,.teaser`

During recursion, matching tags/classes are locked and skipped.

## Article-level exclusions

Typical exclusion controls in `boot.php` include:

- current article is REDAXO notfound article -> always skip
- `articles_exclude` list (comma-separated article IDs)
- `exclude_by_template` (template ID list)
- `exclude_by_meta_field` + `exclude_by_meta_condition` (`<0`, `=0`, `>0`)

This is useful for legal pages, forms, and technical pages where glossary links are undesirable.

## Multi-domain and YRewrite

If YRewrite is active, MultiGlossar can use domain-specific glossary article assignment:

- settings like `article_<domainId>` map each domain to its glossary article
- without YRewrite, fallback is generic `article`

Editorial term storage is not domain-separated; only output linking/context can differ by domain.

## Cache behavior

Important options:

- `use_cache` - use MultiGlossar output cache
- `use_turbocache` - read cached replacement early (`PACKAGES_INCLUDED`)

Cache invalidation hooks are bound to article/category/slice events and glossary form saves. When integrating custom import/update scripts, clear or refresh caches accordingly.

## Practical setup pattern

1. Define a narrow replacement scope around article content.
2. Add ignore tags/classes for navigation, UI components, and structured widgets.
3. Exclude legal/system pages by article ID or metainfo rule.
4. Validate per language and per domain.
5. Enable cache after functional behavior is confirmed.

## Common pitfalls

- Using too broad start/end patterns, replacing content in headers/menus accidentally.
- Forgetting to exclude interactive UI containers, causing tooltip markup conflicts.
- Mixing article IDs from different environments without checking production IDs.
- Using metainfo exclusion with the wrong condition operator (`<0`, `=0`, `>0`).
- Enabling cache during debugging and interpreting stale HTML as parser failure.
