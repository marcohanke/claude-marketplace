---
name: ycom-otp
description: YCom OTP / two-factor authentication setup – TOTP and email-based OTP, the otp_article_id setup article, ycom_auth_otp setup field, OTP email template, enforcement modes, and the injection that gates unverified sessions. Use when the user enables 2FA, configures OTP enforcement levels, builds the OTP setup form, or troubleshoots the OTP flow.
---

# YCom OTP / 2FA

YCom supports two OTP methods: TOTP (authenticator apps like Google Authenticator) and email-delivered codes. Both share the same setup form and injection-based enforcement.

## Setup steps

1. Set `otp_article_id` in YCom settings to point to an OTP setup article.
2. Restrict that article to logged-in users (permission type 1).
3. On the OTP article, add a form with: `ycom_auth_otp|setup`
4. Configure enforcement: `otp_auth_enforce` = `all` or `disabled`.
5. Configure available methods: `otp_auth_option` = `all`, `totp_only`, `email_only`.

## OTP setup form (minimum)

```
ycom_auth_otp|setup
```

That's it. The field renders the QR code (for TOTP), an email code option, and the verification step in one component.

## Configuration keys

| Key | Type | Description |
|---|---|---|
| `otp_article_id` | int | OTP setup/verify article |
| `otp_auth_enforce` | string | `all` or `disabled` |
| `otp_auth_option` | string | `all`, `totp_only`, `email_only` |
| `otp_auth_email_period` | int | OTP email-code period in seconds (300–1800) |

`all` enforcement means every user must complete OTP setup before they can access protected articles. `disabled` makes OTP optional (users can opt-in but no gating).

## Email-based OTP

When `otp_auth_option` allows `email_only` or `all`, YCom sends OTP codes via the email template:

- **Template key:** `ycom_otp_code_template`
- **Available placeholders:** `name`, `email`, `firstname`, `code`

Minimum body:

```
Hallo ###firstname###,

Ihr Anmeldecode lautet: ###code###

Der Code ist 5 Minuten gueltig.
```

Adjust the validity period via `otp_auth_email_period` (300–1800 seconds, default depends on plugin defaults).

## Injection-based enforcement

YCom enforces OTP via `rex_ycom_injection_otp`:

1. After login, the injection checks if the session is OTP-verified.
2. If not, redirect to `otp_article_id`.
3. The user completes setup or enters a fresh code.
4. The session is marked OTP-verified.
5. Next request flows through normally.

The injection is part of the standard auth flow — you don't need to call it manually.

## Skipping OTP for specific roles

When some user roles shouldn't go through OTP:

```php
rex_extension::register('YCOM_AUTH_INIT', function (rex_extension_point $ep) {
    $user = rex_ycom_auth::getUser();
    if ($user && $user->isInGroup($apiOnlyGroupId)) {
        rex_ycom_auth::setSessionVar('otp_verified', true);
    }
});
```

Setting `otp_verified=true` in the session bypasses the OTP injection for that session.

## Common pitfalls

- Setting `otp_auth_enforce=all` without configuring `otp_article_id` – users get stuck in a redirect loop because the destination article doesn't exist.
- The OTP setup article must be readable by logged-in users (permission type 1) – not "all" (3), or unauthenticated users see the form.
- Forgetting to create the `ycom_otp_code_template` email template when using email-based OTP – delivery silently fails.
- TOTP secrets stored in the `rex_ycom_user` table are sensitive – treat the table like password storage; restrict backend access.
- Long `otp_auth_email_period` values combined with shared mailboxes (info@) become a phishing vector – keep periods short.
- Re-using the same code multiple times is rejected by design. If a user reports "code says invalid", ask whether they entered an old code from a previous email.
