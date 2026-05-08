# redaxo-core

Core REDAXO knowledge for Claude Code. Recommended for every REDAXO project.

## What's included

### Skills (auto-activate when relevant)

- **redaxo-architecture** – directory layout, request lifecycle, key classes (`rex_addon`, `rex_config`, `rex_path`, `rex_url`)
- **redaxo-modules** – module input/output PHP, `REX_VALUE`, `REX_MEDIA`, `REX_LINK`, `REX_LINKLIST`, `REX_MEDIALIST` placeholders
- **redaxo-templates** – template structure, `getArticle()`, fragments, multi-language output
- **redaxo-sql-patterns** – `rex_sql` queries, prepared statements, transactions, escaping
- **redaxo-extension-points** – `rex_extension::register`, common EPs, subject/params handling
- **redaxo-addon-development** – `package.yml`, `boot.php`, install/update, lang files, asset pipeline (incl. reinstall-to-sync), cache busting, common pitfalls
- **redaxo-console-commands** – Symfony Console commands for CLI/diagnostic/maintenance tools (`rex_console_command`, `package.yml` registration)
- **redaxo-api-functions** – `rex_api_function` HTTP/AJAX endpoints, `$published` flag, JSON responses, CORS for cross-domain widgets
- **redaxo-metainfo-fields** – metainfo `params` separator rules (`|` vs `:`), idempotent setup, navigation filtering by metainfo

### Slash commands

- `/redaxo-core:rex-create-module` – guided module scaffolding
- `/redaxo-core:rex-create-addon` – guided addon scaffolding

## Install

```bash
/plugin install redaxo-core@redaxo-marketplace
```
