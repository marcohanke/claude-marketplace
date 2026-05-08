# redaxo-search-it

Claude Code support for REDAXO's [Search It](https://github.com/FriendsOfREDAXO/search_it) addon. Search It provides full-text search for articles, media, files (including PDF), URL addon URLs and arbitrary database columns.

## Skills

- **search-it-search** – SearchIt class, executing searches, filters, result array structure, highlighting
- **search-it-modules** – building search form and result modules, pagination, media search, URL addon hits
- **search-it-indexing** – index lifecycle, reindex, console commands, cronjobs, plaintext conversion
- **search-it-configuration** – backend settings, search modes, blacklist, similarity search, autocomplete
- **search-it-extension-points** – SEARCH_IT_INDEX_ARTICLE, SEARCH_IT_PLAINTEXT, SEARCH_IT_SEARCH_EXECUTED

## Install

```bash
/plugin install redaxo-search-it@redaxo-marketplace
```

Make sure the Search It addon is installed, activated and the index has been generated at least once (Backend > Search It > Generate index).
