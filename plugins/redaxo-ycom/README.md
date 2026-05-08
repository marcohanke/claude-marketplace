# redaxo-ycom

Claude Code support for REDAXO's [YCom](https://github.com/yakamara/redaxo_ycom) addon — frontend user management. YCom builds on top of YForm and YRewrite to provide login, registration, password management, article/media permissions, groups, OTP/2FA, tokens, and external auth (SAML/OAuth2/CAS).

## Skills

- **ycom-overview** – core classes, user model (`rex_ycom_user`), status constants, config keys, brute-force rules, auth API
- **ycom-forms** – ready-to-use pipe syntax for login, registration, profile edit, password change/reset, terms-of-use forms
- **ycom-tokens** – token system for direct login, password reset, registration confirmation
- **ycom-permissions** – article permission types, the `group` plugin, the `media_auth` plugin (file protection)
- **ycom-otp** – OTP / two-factor authentication setup and configuration
- **ycom-external-auth** – SAML, OAuth2, CAS configuration and field mapping
- **ycom-extension-points** – YCOM extension points, the injection system, sessions, logging

## Install

```bash
/plugin install redaxo-ycom@redaxo-marketplace
```

YCom requires:

- REDAXO ^5.17
- PHP >= 8.2
- YForm >= 3.2
- YRewrite >= 2.6

Install YCom in the REDAXO backend before using these skills.
