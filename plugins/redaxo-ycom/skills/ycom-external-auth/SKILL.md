---
name: ycom-external-auth
description: External authentication for YCom – SAML, OAuth2, and CAS. Covers config files (data/addons/ycom/saml.php, oauth2.php, cas.php), the ycom_auth_saml/oauth2/cas value fields, default user data, allowed-domain restrictions, and field-mapping via YCOM_AUTH_*_MATCHING extension points. Use when the user integrates SSO, configures OAuth2 providers, maps SAML attributes to YCom user fields, or enables direct-link login.
---

# YCom External Authentication

YCom integrates with three SSO protocols out of the box: SAML, OAuth2, CAS. Each has its own config file, its own value field, and a matching extension point for mapping the external identity to a `rex_ycom_user`.

## SAML

Config file: `data/addons/ycom/saml.php` — return an array of SAML settings consumed by the underlying SimpleSAMLphp / OneLogin library YCom uses.

Login button on a YForm form:

```
ycom_auth_saml|SAML Login|Fehler bei SAML-Login|example.com|{"status":1}|1
```

Parameters:
- `label` – button label
- `error_msg` – shown when SAML returns an error
- `allowed_domains` – comma-separated email domains allowed to register via SAML (empty = all)
- `default_userdata_json` – JSON with default field values for newly-created users (e.g. `{"status":1}`)
- `direct_link` (`0/1`) – if `1`, redirect to the IdP immediately on page load instead of showing a click-to-login button

## OAuth2

Config file: `data/addons/ycom/oauth2.php` — return a settings array consumed by the underlying OAuth2 client.

```php
<?php
$settings = [
    'clientId'                => 'myapp',
    'clientSecret'            => 'secret',
    'redirectUri'             => '',                                      // empty = use current URL
    'urlAuthorize'            => 'https://provider.com/authorize',
    'urlAccessToken'          => 'https://provider.com/token',
    'urlResourceOwnerDetails' => 'https://provider.com/userinfo',
];
return $settings;
```

Login button:

```
ycom_auth_oauth2|OAuth2 Login|Fehler bei OAuth2-Login|example.com|{"status":1}|1
```

Same parameter shape as SAML.

## CAS

Config file: `data/addons/ycom/cas.php` — settings for the CAS server.

```
ycom_auth_cas|CAS Login|Fehler bei CAS-Login|example.com|{"status":1}
```

CAS does not support `direct_link` (no parameter) — always click-to-login.

## Field mapping (matching external identity to YCom user)

When an external login completes, YCom creates or updates a `rex_ycom_user`. The mapping is customizable via per-protocol extension points:

```php
rex_extension::register('YCOM_AUTH_SAML_MATCHING', function (rex_extension_point $ep) {
    $params   = $ep->getParams();
    $userdata = $ep->getSubject();

    // Map external attributes onto YCom user fields
    $userdata['firstname'] = $params['saml_attributes']['givenName'][0] ?? '';
    $userdata['name']      = $params['saml_attributes']['surname'][0]   ?? '';
    $userdata['email']     = $params['saml_attributes']['mail'][0]      ?? '';

    return $userdata;
});
```

OAuth2 and CAS use the same pattern with `YCOM_AUTH_OAUTH2_MATCHING` / `YCOM_AUTH_CAS_MATCHING`.

There's also a generic `YCOM_AUTH_MATCHING` that fires for all external auth methods — use it for shared logic (e.g. "set group based on email domain") and the per-protocol EPs for protocol-specific attribute mapping.

## Allowed domains

The `allowed_domains` parameter on the value field is a comma-separated list of email domains. Logins from emails outside the list are rejected even if the IdP succeeds — useful for B2B portals where only employees of partner companies should be allowed.

```
ycom_auth_saml|SAML Login|Fehler|partner1.com,partner2.com|{"status":1}|0
```

Empty domain list = allow all.

## Default user data

`default_userdata_json` sets values on new users created via the external auth. Common patterns:

```json
{"status": 1}
```

Auto-confirm new SSO users (skip the registration confirmation flow — the IdP already verified them).

```json
{"status": 1, "ycom_groups": "5"}
```

Auto-confirm AND assign group 5 (e.g. "SSO users").

## Direct link mode

`direct_link=1` skips the click-to-login button and redirects to the IdP immediately when the form loads. Useful for SSO-only sites where there's no email/password fallback.

When using `direct_link`, make sure the form is on a dedicated login article — placing it on a generic page would auto-redirect every visitor.

## Common pitfalls

- Forgetting `default_userdata_json` for SAML/OAuth2 users — new users default to `status=0` (pending) and can't actually log in until manually confirmed.
- The `redirectUri` in OAuth2 settings must match what's registered with the provider — including trailing slash differences.
- Mapping the `email` field but not `login` – YCom uses `login` for the unique identifier when `login_field=login`. Either set both or change `login_field` to `email`.
- Per-protocol matching EPs run AFTER the generic `YCOM_AUTH_MATCHING`. If both modify the same field, the protocol-specific one wins.
- CAS doesn't expose attributes the way SAML/OAuth2 do – the matching params are sparse. Plan your `default_userdata_json` accordingly.
- SAML config files are PHP — do not commit them with real secrets. Use environment variables and load them from a non-tracked file.
