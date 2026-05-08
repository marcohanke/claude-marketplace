---
name: yrewrite-seo
description: SEO helpers in YRewrite – meta tags via rex_yrewrite_seo, sitemap.xml generation, robots.txt, canonical and hreflang tags. Use when the user adds SEO meta to templates, configures sitemap inclusion, customizes robots.txt, or sets up structured social-share metadata (Open Graph, Twitter cards).
---

# YRewrite SEO

YRewrite ships with `rex_yrewrite_seo` – a helper that pulls SEO data from articles (title, description, image) and renders meta tags. It also generates `sitemap.xml` and a `robots.txt` block.

## Article SEO data

Each article has SEO fields managed via the `seo` tab in the backend (article edit). They live as meta-info fields:

- `art_yrewrite_title` – `<title>` override (falls back to article name)
- `art_yrewrite_description` – meta description
- `art_yrewrite_index` – `index`, `noindex`, or empty (default = follow site setting)
- `art_yrewrite_canonical_url` – override canonical
- `art_yrewrite_image` – Open Graph image filename

For domain-level defaults (site title, description), set them per domain in YRewrite → Domains → SEO.

## Rendering tags in the template

Place this in the `<head>` of your template:

```php
<?php $seo = new rex_yrewrite_seo(); ?>
<title><?= rex_escape($seo->getTitle()) ?></title>
<meta name="description" content="<?= rex_escape($seo->getDescription(), 'html_attr') ?>">
<?= $seo->getRobotsTag() ?>
<?= $seo->getCanonicalUrlTag() ?>
<?= $seo->getHreflangTags() ?>
```

`getRobotsTag()`, `getCanonicalUrlTag()`, and `getHreflangTags()` produce full `<meta>` / `<link>` elements. They're already escaped – don't wrap them in `rex_escape`.

## Open Graph & Twitter cards

YRewrite doesn't render OG tags itself. Add them yourself using the SEO data plus an explicit OG image:

```php
<?php
$seo = new rex_yrewrite_seo();
$ogImage = (string) rex_article::getCurrent()->getValue('art_yrewrite_image');
$ogImageUrl = $ogImage
    ? rex::getServer() . rex_url::media($ogImage)
    : rex::getServer() . rex_url::frontend('assets/og-default.jpg');
?>
<meta property="og:title" content="<?= rex_escape($seo->getTitle(), 'html_attr') ?>">
<meta property="og:description" content="<?= rex_escape($seo->getDescription(), 'html_attr') ?>">
<meta property="og:url" content="<?= rex_escape(rex_yrewrite::getFullUrlByArticleId(rex_article::getCurrentId(), rex_clang::getCurrentId()), 'html_attr') ?>">
<meta property="og:type" content="website">
<meta property="og:image" content="<?= rex_escape($ogImageUrl, 'html_attr') ?>">

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="<?= rex_escape($seo->getTitle(), 'html_attr') ?>">
<meta name="twitter:description" content="<?= rex_escape($seo->getDescription(), 'html_attr') ?>">
<meta name="twitter:image" content="<?= rex_escape($ogImageUrl, 'html_attr') ?>">
```

## sitemap.xml

YRewrite auto-generates `https://<domain>/sitemap.xml`. To exclude an article, set `art_yrewrite_index` to `noindex` or check "exclude from sitemap" in the backend.

To extend the sitemap with dynamic entries (e.g. YForm dataset URLs), hook into the sitemap generation:

```php
// In your addon's boot.php
rex_extension::register('YREWRITE_SITEMAP', function (rex_extension_point $ep) {
    $items = $ep->getSubject();

    foreach (team_member::query()->where('status', 1)->find() as $member) {
        $items[] = [
            'loc'        => rex_yrewrite::getFullUrlByArticleId(42, rex_clang::getCurrentId())
                          . '?member=' . $member->getId(),
            'lastmod'    => date('c', strtotime((string) $member->getValue('updatedate'))),
            'changefreq' => 'monthly',
            'priority'   => '0.6',
        ];
    }

    return $items;
});
```

## robots.txt

YRewrite serves `robots.txt` per domain. Configure each domain's robots block in YRewrite → Domains → robots.txt. Default content:

```
User-agent: *
Allow: /
Sitemap: https://www.example.com/sitemap.xml
```

For staging environments, swap to:

```
User-agent: *
Disallow: /
```

Detect environment in your template or `boot.php` and switch with an extension point if you don't want to maintain separate domain configs.

## Multi-language SEO checklist

- ✅ `<html lang="...">` matches the active clang code
- ✅ Self-canonical with full URL (already handled by `getCanonicalUrlTag()`)
- ✅ `hreflang` for every supported language including `x-default` (`getHreflangTags()` does this)
- ✅ Translated `<title>` and meta description per article per language
- ✅ Domain-specific sitemap (don't merge sitemaps across domains)

## Structured data (JSON-LD)

YRewrite doesn't render JSON-LD; add it explicitly. For an article page:

```php
<script type="application/ld+json">
<?= json_encode([
    '@context'      => 'https://schema.org',
    '@type'         => 'Article',
    'headline'      => $seo->getTitle(),
    'description'   => $seo->getDescription(),
    'datePublished' => date('c', rex_article::getCurrent()->getCreateDate()),
    'dateModified'  => date('c', rex_article::getCurrent()->getUpdateDate()),
    'author'        => [
        '@type' => 'Organization',
        'name'  => rex::getServerName(),
    ],
    'mainEntityOfPage' => rex_yrewrite::getFullUrlByArticleId(
        rex_article::getCurrentId(),
        rex_clang::getCurrentId()
    ),
], JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE) ?>
</script>
```

## Common pitfalls

- Wrapping `getRobotsTag()` / `getHreflangTags()` in `rex_escape()` – they output safe markup already. Double-escaping breaks them.
- Forgetting OG images on social posts – Twitter and LinkedIn fall back to no image, hurting click-throughs.
- Setting `art_yrewrite_canonical_url` to a duplicate URL on multiple articles – Google then ignores all but one.
- Adding articles to the sitemap that return 404 / 403 – costs crawl budget. Run a periodic sitemap audit.
- Leaving `Disallow: /` in production after copying from staging.
