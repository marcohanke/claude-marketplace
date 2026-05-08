# redaxo-yform

Claude Code support for REDAXO's [YForm](https://github.com/yakamara/redaxo_yform) addon. YForm adds dynamic forms, custom database tables, an ORM-style dataset API, validation, email templates, and a REST API layer.

## Skills

- **yform-tables** – defining tables in `install.php`, the Table Manager UI, schema migrations, tableset import/export
- **yform-fields** – complete field/validate/action reference, choice/relation/upload configs, custom field types, pipe syntax
- **yform-frontend** – building public-facing forms (contact, registration, edit), CSRF, file uploads, spam protection, objparams
- **yform-datasets** – `rex_yform_manager_dataset` (YOrm) – queries, joins, pagination, relations, dataset-driven forms
- **yform-email-templates** – placeholder syntax (`REX_YFORM_DATA[field=...]`), `tpl2email`, sending programmatically, attachments
- **yform-rest-api** – exposing YForm tables as JSON:API endpoints with token auth, CORS, per-method field whitelists

## Install

```bash
/plugin install redaxo-yform@redaxo-marketplace
```

Make sure the YForm addon is installed and activated in your REDAXO backend before using these skills.

## Requirements

- REDAXO ^5.17 (some skills assume ^5.18 features)
- PHP >= 8.1
- YForm 4.x recommended; 3.x supported with reduced REST API capabilities
