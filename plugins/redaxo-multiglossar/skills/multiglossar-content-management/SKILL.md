---
name: multiglossar-content-management
description: MultiGlossar backend data management - creating terms and term_alt aliases, activating/deactivating terms, multi-language behavior with clang tabs, and editor permissions. Use when the user works with pages/main.php, rex_multiglossar records, clang-specific glossary entries, or asks how terms are maintained across languages.
---

# MultiGlossar Content Management

MultiGlossar stores glossary terms in `rex_multiglossar` and lets editors maintain central definitions that are injected into frontend content.

## Core data model

Typical fields in glossary records:

- `pid` - shared logical term ID across languages
- `clang_id` - language assignment
- `term` - main glossary term
- `term_alt` - additional terms/aliases (newline-separated)
- `definition` - short definition text used in replacement output
- `active` - whether the term is currently used for replacement
- `casesensitive` - case-sensitive matching toggle

A term is conceptually language-grouped by `pid`, while language-specific content is held per `clang_id` row.

## Backend workflow

In `multiglossar/main`, editors usually:

- add a new term (initially disabled by default)
- enter definition and optional aliases in `term_alt`
- activate/deactivate terms depending on release state
- sort/search rows for maintenance

Because replacements run in frontend output, even small text changes should be treated as content release changes.

## Multi-language behavior

MultiGlossar is designed for multi-language REDAXO setups:

- New terms are created for all languages.
- Deleting a term removes all language variants of that term.
- If a language is added later, terms from the start language are copied and set inactive.
- If a language is deleted, related glossary rows are removed.

This means editors should coordinate glossary activation per language, especially on partially translated projects.

## Permissions and editorial governance

By convention in MultiGlossar:

- Add/delete actions are restricted to admins or users with broad language permissions.
- Translators without global language permissions should focus on content updates, not structure changes.

When implementing custom backend tooling, keep these permission boundaries aligned with `rex::getUser()` and clang permissions.

## Integration with glossary article/detail page

Many projects use one dedicated glossary article as detail target, while in-text replacements remain inline tooltip/help markers. Ensure all terms link consistently to the chosen glossary article structure.

## Common pitfalls

- Treating one language row as fully independent: rows are linked by `pid`, so structural actions affect all languages.
- Storing multiple aliases comma-separated instead of newline-separated in `term_alt`, which can break expected matching behavior.
- Activating copied fallback entries for new languages too early before translations are ready.
- Allowing unrestricted delete access for editors, causing cross-language data loss.
