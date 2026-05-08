---
name: ycom-overview
description: YCom architecture for REDAXO – core classes, user model (rex_ycom_user), authentication API (rex_ycom_auth), status constants, config keys (article_id_login, session_duration, otp_*), brute-force rules, and the standard navigation/login-state integration. Use when the user works with YCom for the first time, looks up an auth API method, configures session timeouts or login redirects, builds a meta navigation that switches based on login state, or programmatically creates/logs in users.
---

# YCom Overview

YCom is REDAXO's frontend user-management addon. It plugs into YForm (for forms/datasets) and YRewrite (for permissions on pretty URLs). The result: registration, login, sessions, group permissions, article gating, OTP, external auth — all configured in the backend, consumed in templates.

## Plugins shipped with YCom

| Plugin | Purpose |
|---|---|
| `auth` | Login/logout, sessions, article permissions, brute-force protection, OTP, injections |
| `group` | User groups + group-based article/media permissions |
| `media_auth` | Media-pool file protection with auth/group checks |

## Core classes

| Class | Purpose |
|---|---|
| `rex_ycom` | Base class, table registry |
| `rex_ycom_config` | Config access |
| `rex_ycom_user` | User model (extends `rex_yform_manager_dataset`) |
| `rex_ycom_auth` | Authentication engine, login/logout, permission checks |
| `rex_ycom_auth_rules` | Brute-force protection rules |
| `rex_ycom_user_session` | Session management (DB-backed) |
| `rex_ycom_user_token` | Token generation/validation |
| `rex_ycom_group` | Group model and permission checks |
| `rex_ycom_media_auth` | Media file permission checks |
| `rex_ycom_log` | User activity logging |

## User model (`rex_ycom_user`)

`rex_ycom_user` extends `rex_yform_manager_dataset` — use the YOrm methods (see the `yform-datasets` skill).

### Status constants

| Constant | Value | Meaning |
|---|---|---|
| `STATUS_INACTIVE_TERMINATION` | -3 | Terminated |
| `STATUS_INACTIVE_LOGINS` | -2 | Deactivated (too many failed logins) |
| `STATUS_INACTIVE` | -1 | Inactive |
| `STATUS_REQUESTED` | 0 | Requested (pending confirmation) |
| `STATUS_CONFIRMED` | 1 | Confirmed and active |
| `STATUS_ACTIVE` | 2 | Active |

Login allowed: `status >= 1`. Login denied: `status <= -1`. Pending: `status = 0`.

### Common methods

```php
// Get current logged-in user
$user = rex_ycom_auth::getUser(); // rex_ycom_user|null

// Field access
$user->getValue('login');
$user->getValue('email');
$user->getValue('firstname');
$user->getValue('name');
$user->getValue('status');
$user->getValue('last_login_time');
$user->getValue('last_action_time');
$user->getValue('ycom_groups');

// Group checks
$user->isInGroup($group_id); // bool
$user->getGroups();          // array of group IDs

// Related groups (YOrm collection)
foreach ($user->getRelatedCollection('ycom_groups') as $group) {
    echo $group->getValue('name');
}

// Create user programmatically
rex_ycom_user::createUserByEmail([
    'email'     => 'user@example.com',
    'login'     => 'user@example.com',
    'password'  => $hashedPassword,
    'firstname' => 'Max',
    'name'      => 'Mustermann',
    'status'    => rex_ycom_user::STATUS_CONFIRMED,
]);

// Update current user
rex_ycom_user::updateUser(['firstname' => 'New Name']);
```

### Extending status options

```php
rex_extension::register('YCOM_USER_STATUS_OPTIONS', function (rex_extension_point $ep) {
    $options = $ep->getSubject();
    $options[3] = 'translate:ycom_account_premium';
    ksort($options);
    return $options;
});
```

## Authentication API (`rex_ycom_auth`)

### Auth status constants

| Constant | Value |
|---|---|
| `STATUS_NOT_LOGGED_IN` | 0 |
| `STATUS_IS_LOGGED_IN` | 1 |
| `STATUS_HAS_LOGGED_IN` | 2 |
| `STATUS_HAS_LOGGED_OUT` | 3 |
| `STATUS_LOGIN_FAILED` | 4 |

### Article permission types

| Value | Meaning |
|---|---|
| 0 | Inherit from parent (default: accessible to all) |
| 1 | Only logged-in users (+ optional group checks) |
| 2 | Only NOT logged-in users |
| 3 | Accessible to all |

### Common auth methods

```php
$user = rex_ycom_auth::getUser();

if (rex_ycom_auth::getUser()) { /* logged in */ }

// Check article permission
$article = rex_article::get($id);
rex_ycom_auth::articleIsPermitted($article); // bool

// Programmatic login (e.g. after a custom registration step).
// Each key/value becomes a YOrm where() clause — use real columns,
// not magic names. `login` is a string column, NOT a user id.
rex_ycom_auth::loginWithParams(['id' => $userId]);
rex_ycom_auth::loginWithParams(['email' => $email]);

// Logout (writes log entry, clears stay-logged-in cookie, clears session)
rex_ycom_auth::logout($user);

// Low-level: clear session only (no log entry, no cookie cleanup)
rex_ycom_auth::clearUserSession();

// Session variables
rex_ycom_auth::setSessionVar('key', $value);
rex_ycom_auth::getSessionVar('key', 'string', $default);
rex_ycom_auth::unsetSessionVar('key');

// Check password
rex_ycom_auth::checkPassword($password, $userId); // bool

// Delete user
rex_ycom_auth::deleteUser($id);
```

### Login flow (`rex_ycom_auth::init()`)

1. Checks credentials/session/cookie for login.
2. On `STATUS_HAS_LOGGED_IN`: redirect to `article_id_jump_ok`.
3. Checks article permissions via `articleIsPermitted()`.
4. Not permitted + not logged in: redirect to `article_id_login`.
5. Not permitted + logged in: redirect to `article_id_jump_denied`.
6. On `STATUS_HAS_LOGGED_OUT`: redirect to `article_id_jump_logout`.
7. Processes injections (OTP, password change, terms of use).

### Navigation integration

```php
$nav = rex_navigation::factory();
$nav->addCallback('rex_ycom_auth::articleIsPermitted');
echo $nav->show(0, 1, true, true);
```

This filters out articles the current user isn't allowed to see, with no template changes.

### Meta navigation (login/logout button)

```php
$user = rex_ycom_auth::getUser();
$login_article  = rex_ycom_config::get('article_id_login');
$logout_article = rex_ycom_config::get('article_id_logout');

if ($user) {
    echo '<a href="' . rex_getUrl($profile_id) . '">'
         . rex_escape($user->getValue('firstname')) . '</a>';
    echo '<a href="' . rex_getUrl($logout_article) . '">Logout</a>';
} else {
    echo '<a href="' . rex_getUrl($login_article) . '">Login</a>';
    echo '<a href="' . rex_getUrl($register_id) . '">Registrieren</a>';
}
```

## Configuration keys (`rex_ycom_config::get()`)

Override via the `YCOM_CONFIG` extension point.

| Key | Type | Description |
|---|---|---|
| `article_id_login` | int | Login page article |
| `article_id_logout` | int | Logout page article |
| `article_id_register` | int | Registration page |
| `article_id_password` | int | Password reset page |
| `article_id_jump_ok` | int | Redirect after successful login |
| `article_id_jump_not_ok` | int | Redirect after failed login (external auth) |
| `article_id_jump_logout` | int | Redirect after logout |
| `article_id_jump_denied` | int | Redirect when no group permission |
| `article_id_jump_password` | int | Forced password change article |
| `article_id_jump_termsofuse` | int | Terms of use acceptance article |
| `auth_rule` | string | Brute-force rule (see below) |
| `auth_cookie_ttl` | int | Stay-logged-in cookie TTL in days (0/7/14/30/90/180/360) |
| `login_field` | string | `email` or `login` |
| `session_duration` | int | Inactivity timeout in seconds (default 3600) |
| `session_max_overall_duration` | int | Max session lifetime in seconds (default 21600) |
| `otp_article_id` | int | OTP setup/verify article |
| `otp_auth_enforce` | string | `all` or `disabled` |
| `otp_auth_option` | string | `all`, `totp_only`, `email_only` |
| `otp_auth_email_period` | int | OTP email-code period in seconds (300–1800) |

## Brute-force rules

| Rule | Behavior |
|---|---|
| `login_try_unlimited` | No limit |
| `login_try_5_deactivate` | Deactivate user after 5 failed tries |
| `login_try_10_deactivate` | Deactivate after 10 |
| `login_try_20_deactivate` | Deactivate after 20 |
| `login_try_5_pause` | 15-min pause every 5th failed try |
| `login_try_10_pause` | 15-min pause every 10th failed try |
| `login_try_10_always_pause` | 5-min pause after 10+ tries |

Set via `auth_rule` in YCom settings.

## Common pitfalls

- Treating `rex_ycom_auth::getUser()` as a guaranteed user object — it returns `null` when nobody is logged in. Always check.
- Reading status with `==` against `STATUS_REQUESTED` (0) without checking for false-positives — `null == 0` is true in loose comparison. Use `===` or constants.
- Forgetting that `articleIsPermitted()` doesn't redirect on its own — it just returns bool. The redirect happens inside `rex_ycom_auth::init()`.
- Hardcoding article IDs in templates — always pull from `rex_ycom_config::get('article_id_login')` etc., so the editor can change them in the backend.
- Calling `loginWithParams()` from a context where the response was already sent — login sets headers (cookie + redirect), must run before output.
- Passing a user id as `['login' => $userId]` to `loginWithParams()` — `login` is the string login name, not the id. Use `['id' => $userId]` or `['email' => $email]`.
- Using `clearUserSession()` as the user-facing logout — it skips the log entry and stay-logged-in cookie cleanup. Use `rex_ycom_auth::logout($user)` for the full flow.
