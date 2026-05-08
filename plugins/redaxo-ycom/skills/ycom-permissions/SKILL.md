---
name: ycom-permissions
description: Article and media permissions in YCom – the group plugin (group-based article/media access, group permission types) and the media_auth plugin (media-pool file protection, .htaccess rewrite rule, redirect rules). Use when the user gates articles by login state or group, sets up media file protection, configures redirects for unauthorized access, or queries group membership programmatically.
---

# YCom Permissions

Permissions in YCom come in two layers: per-article (via the `auth` plugin) and per-media-file (via the `media_auth` plugin). Both can be tightened by group membership when the `group` plugin is active.

## Article permissions (built into `auth`)

Each article has a permission type set in the structure backend:

| Value | Meaning |
|---|---|
| 0 | Inherit from parent (default: accessible to all) |
| 1 | Only logged-in users (+ optional group checks) |
| 2 | Only NOT logged-in users (e.g. registration page) |
| 3 | Accessible to all (overrides parent) |

YCom intercepts the request in `rex_ycom_auth::init()`:

- Not permitted + not logged in → redirect to `article_id_login`
- Not permitted + logged in → redirect to `article_id_jump_denied`

To check programmatically:

```php
$article = rex_article::get($id);
rex_ycom_auth::articleIsPermitted($article); // bool
```

Hide gated articles from navigation:

```php
$nav = rex_navigation::factory();
$nav->addCallback('rex_ycom_auth::articleIsPermitted');
echo $nav->show(0, 1, true, true);
```

## Group plugin

Groups add finer-grained permissions on top of "logged in or not".

### Group permission types (per article)

| Value | Meaning |
|---|---|
| 0 | Accessible for all groups |
| 1 | User must be in ALL specified groups |
| 2 | User must be in at least ONE specified group |
| 3 | User must have NO groups |

### Group API

```php
// Get all groups
$groups = rex_ycom_group::getGroups(); // [id => name]

// Check user group membership
$user->isInGroup($group_id);

// Get users in a group
$users = rex_ycom_group::get($id)->getRelatedCollection('ycom_groups');

// Get user's groups as YOrm collection
foreach ($user->getRelatedCollection('ycom_groups') as $group) {
    echo $group->getValue('name');
}
```

### Group-based redirect after login

```php
$user = rex_ycom_auth::getUser();
if ($user) {
    $group = $user->getRelatedDataset('ycom_groups');
    if ($group) {
        $target = $group->getValue('target_id'); // custom be_link field on the group table
        if ($target) {
            rex_response::sendRedirect(rex_getUrl($target));
        }
    }
}
```

Add a `target_id` field (type `be_link`) to the YCom group table so editors can configure where each group lands after login.

### Extending article permission UI

```php
rex_extension::register('YCOM_ARTICLE_PERM_SELECT', function (rex_extension_point $ep) {
    // Customize the permission select dropdown shown in the structure sidebar
});
```

## Media auth plugin

Protects files in the REDAXO media pool from direct download.

### Setup

1. Install YRewrite (`media_auth` depends on it).
2. Activate the `media_auth` plugin.
3. Configure rules in plugin settings.
4. Add the rewrite rule to `.htaccess`:

```apache
RewriteRule ^media/(.*) %{ENV:BASE}/index.php?rex_media_type=default&rex_media_file=$1&%{QUERY_STRING} [B]
```

This routes every `/media/...` request through `index.php`, where YCom can decide whether to serve the file.

### Per-media permission types

Each media file gets a permission type set in the media pool:

| Value | Meaning |
|---|---|
| 0 | Accessible to all |
| 1 | Only logged-in users |

For group-scoped media access, combine with the `group` plugin.

### Auth rules (what happens when access is denied)

| Rule | Behavior |
|---|---|
| `redirect` | Redirect to login page |
| `redirect_with_errorpage` | Redirect to login if not logged in, error page if already logged in |
| `header_notfound` | Return 404 |
| `header_perm_denied` | Return 401 |

`header_notfound` is the most discreet — leaks no information about the file's existence.

### Programmatic media check

```php
$media = rex_media::get('filename.pdf');
rex_ycom_media_auth::checkFrontendPerm($media); // bool
```

Use this when you build a custom download endpoint and need to verify access manually.

## Common pitfalls

- Setting article permission to "logged-in only" but forgetting that the login page itself must be set to `2` (only NOT logged in) — otherwise logged-in users get bounced to the post-login redirect when they navigate to it.
- Forgetting the `.htaccess` rewrite for media_auth — the plugin is active but files are still served directly by Apache, bypassing checks.
- Using `header_notfound` for files that should be discoverable to logged-in users — the 404 doesn't differentiate; users get told the file doesn't exist instead of "log in to access".
- Group ID 0 is "no group". Permissions of type 1 (must be in ALL) with no specified groups always pass. Type 2 (must be in ONE) with no specified groups always fails.
- Calling `rex_ycom_media_auth::checkFrontendPerm()` from a backend context — it checks the frontend session, returns false in backend context. Use a backend-specific check instead.
