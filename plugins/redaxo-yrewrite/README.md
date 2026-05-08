# redaxo-yrewrite

Claude Code support for REDAXO's [YRewrite](https://github.com/yakamara/redaxo_yrewrite) addon. YRewrite adds pretty URLs, multi-domain hosting, redirects, and SEO helpers (meta tags, sitemap, robots.txt).

## Skills

- **yrewrite-domains** – domain configuration, language mapping, fetching URLs
- **yrewrite-redirects** – `rex_yrewrite_forward`, `.htaccess` rules, 301/302 strategies
- **yrewrite-seo** – meta tags, sitemap.xml, robots.txt, canonical/hreflang

## Install

```bash
/plugin install redaxo-yrewrite@redaxo-marketplace
```

Make sure the YRewrite addon is installed and your webserver rewrites are configured (a working `.htaccess` for Apache, the equivalent `try_files` block for nginx).
