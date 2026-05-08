---
name: ycom-forms
description: Ready-to-use YCom form recipes – login, logout, registration with email confirmation, profile edit, password change (with and without old-password check), password reset, terms-of-use re-acceptance. Pipe and PHP syntax. Includes the YCom-specific value/validate/action fields (ycom_auth_password, ycom_auth_load_user, ycom_auth_logout, ycom_auth_returnto, ycom_auth_db, ycom_user_token, etc.). Use when the user builds any frontend form that interacts with YCom.
---

# YCom Forms

YCom registers its own YForm value/validate/action fields. Combined with a configured login/register/password article, the standard auth flows reduce to a handful of pipe-syntax lines.

## YCom-specific YForm fields (cheat sheet)

### Value fields

```
ycom_auth_password|name|label|password_rules_json|message|[script 0/1]
ycom_auth_load_user|label|opt:field1,field2,field3
ycom_auth_logout|label|[allowed_domains]|[returnTo]
ycom_auth_returnto|label|[allowed_domains]|[fixed_url]
ycom_auth_otp|setup
ycom_auth_saml|label|error_msg|[allowed_domains]|default_userdata_json|[direct_link 0/1]
ycom_auth_oauth2|label|error_msg|[allowed_domains]|default_userdata_json|[direct_link 0/1]
ycom_auth_cas|label|error_msg|[allowed_domains]|[default_userdata_json]
ycom_user_token|token|create|type|email_field
ycom_user_token|token|validate|type|error_message
ycom_user|label|dbfield|fieldlabel|hidden|[no_db]|showlabel
ycom_user_init|[label]|[name]|error_msg
```

### Validate fields

```
validate|ycom_auth|loginfield|passwordfield|stayfield|msg_empty|msg_failed
validate|ycom_auth_login|field1=request1,field2=request2|status_condition|msg|opt:load_fields|[no_db]
validate|ycom_auth_password|passwordfield|message
```

### Action fields

```
action|ycom_auth_db              -- update current user
action|ycom_auth_db|update       -- explicit update
action|ycom_auth_db|delete       -- delete current user
```

### Password rules JSON (`ycom_auth_password`)

```json
{
    "length": {"min": 10},
    "letter": {"min": 1, "generate": 10},
    "lowercase": {"min": 0},
    "uppercase": {"min": 0},
    "digit": {"min": 1},
    "symbol": {"min": 0}
}
```

## Login form

### Pipe

```
validate|ycom_auth|login|password|stayfield|Bitte Login und Passwort eingeben|Login fehlgeschlagen
text|login|Benutzername|||{"autocomplete":"username"}
password|password|Passwort|||{"autocomplete":"current-password"}
checkbox|stayfield|Eingeloggt bleiben
ycom_auth_returnto|returnTo|
```

### PHP

```php
$form = rex_yform::factory();
$form->setObjectparams('form_name', 'login_form');
$form->setObjectparams('form_action', rex_getUrl());

$form->setValidateField('ycom_auth', [
    'login', 'password', null,
    'Bitte Login und Passwort eingeben',
    'Login fehlgeschlagen',
]);
$form->setValueField('text',     ['login', 'Benutzername', '', '', '{"autocomplete":"username"}']);
$form->setValueField('password', ['password', 'Passwort', '', '', '{"autocomplete":"current-password"}']);
$form->setValidateField('empty', ['login', 'Bitte Benutzernamen eingeben']);
$form->setValidateField('empty', ['password', 'Bitte Passwort eingeben']);
$form->setValueField('ycom_auth_returnto', ['returnTo']);
echo $form->getForm();
```

`ycom_auth_returnto` preserves a `returnTo` URL across the login flow — useful when a user clicked a protected link, was redirected to login, and should land back on the original article afterwards.

## Logout form

```
ycom_auth_logout|label|
```

That's the entire form. Submitting it logs the user out and applies the configured `article_id_jump_logout` redirect.

## Profile edit form

```
ycom_auth_load_user|userinfo|email,firstname,name
objparams|form_showformafterupdate|1
showvalue|email|E-Mail-Adresse / Login
text|firstname|Vorname
validate|empty|firstname|Bitte Vornamen eingeben.
text|name|Nachname
validate|empty|name|Bitte Nachnamen eingeben.
action|showtext|<div class="alert alert-success">Profildaten aktualisiert</div>|||1
action|ycom_auth_db
```

`ycom_auth_load_user` pre-fills the form with the current user's values for the listed fields. `action|ycom_auth_db` writes back to the user record.

## Registration with email confirmation

### Step 1 – the registration form

```
generate_key|activation_key
hidden|status|0
fieldset|label|Login-Daten:
text|email|E-Mail*
text|email_2|E-Mail bestaetigen*||no_db
text|firstname|Vorname*
validate|empty|firstname|Bitte Vornamen eingeben.
text|name|Nachname*
validate|empty|name|Bitte Nachnamen eingeben.
ycom_auth_password|password|Passwort*|{"length":{"min":10},"letter":{"min":1},"digit":{"min":1}}|Passwort muss mind. 10 Zeichen und eine Ziffer enthalten.
password|password_2|Passwort bestaetigen*||no_db|{"autocomplete":"new-password"}
checkbox|termsofuse_accepted|Ich akzeptiere die Nutzungsbedingungen.|0|0|
html|required|<p class="form-required">* Pflichtfelder</p>
validate|type|email|email|Bitte gueltige E-Mail eingeben.
validate|unique|email|E-Mail wird bereits verwendet.|rex_ycom_user
validate|empty|password|Bitte Passwort eingeben.
validate|compare|password|password_2||Bitte zweimal dasselbe Passwort eingeben.
validate|compare|email|email_2||Bitte zweimal dieselbe E-Mail eingeben.
action|copy_value|email|login
action|db|rex_ycom_user
action|tpl2email|access_request_de|email|
```

### Step 2 – the confirmation email template

```php
<?php
$article_id = 999; // ID of the confirmation article (Step 3)
$url = rex_getUrl($article_id, '', [
    'rex_ycom_activation_key' => 'REX_YFORM_DATA[field=activation_key]',
    'rex_ycom_id'             => 'REX_YFORM_DATA[field=email]',
]);
$full_url = trim(rex::getServer(), '/') . trim($url, '.');
?>
<p>Bitte klicken Sie diesen Link zur Bestaetigung:</p>
<p><a href="<?= $full_url ?>"><?= $full_url ?></a></p>
```

### Step 3 – the confirmation article

```
hidden|status|1
objparams|submit_btn_show|0
objparams|send|1
objparams|csrf_protection|0
validate|ycom_auth_login|activation_key=rex_ycom_activation_key,email=rex_ycom_id|status=0|Zugang bereits bestaetigt oder fehlgeschlagen|status
action|ycom_auth_db|update
action|html|<b>Vielen Dank, Ihre E-Mail wurde bestaetigt.</b>
```

`csrf_protection|0` is required because the confirmation link can't carry a CSRF token. `validate|ycom_auth_login` looks up the user by `activation_key + email` and gates on the current status (0 = pending).

## Password change form

### Without old-password check

```
ycom_auth_password|password|Neues Passwort*|{"length":{"min":10},"letter":{"min":1},"digit":{"min":1}}|Passwort-Anforderungen nicht erfuellt.
password|password_2|Passwort wiederholen||no_db
validate|empty|password|Bitte Passwort eingeben.
validate|compare|password|password_2|!=|Bitte zweimal dasselbe Passwort eingeben.
action|showtext|Passwort wurde aktualisiert.|||1
action|ycom_auth_db
hidden|new_password_required|0
```

### With old-password check

```
password|old_password|Bisheriges Passwort||no_db|{"autocomplete":"current-password"}
validate|empty|old_password|Bitte bisheriges Passwort angeben.
validate|ycom_auth_password|old_password|Bisheriges Passwort ist nicht korrekt.
ycom_auth_password|password|Neues Passwort*|{"length":{"min":10},"letter":{"min":1},"digit":{"min":1}}|Passwort-Anforderungen nicht erfuellt.
password|password_2|Passwort wiederholen||no_db
validate|empty|password|Bitte Passwort eingeben.
validate|compare|password|password_2|!=|Passwoerter stimmen nicht ueberein.
validate|compare|password|old_password|==|Neues Passwort darf nicht dem bisherigen entsprechen.
action|showtext|Passwort wurde aktualisiert.|||1
action|ycom_auth_db
hidden|new_password_required|0
```

`new_password_required|0` clears the YCom flag that forces password change on next login (used by the password-change injection).

## Password reset

### Reset request

```
generate_key|activation_key
text|email|E-Mail:
validate|type|email|email|Bitte gueltige E-Mail eingeben.
validate|empty|email|Bitte E-Mail eingeben.
validate|in_table|email|rex_ycom_user|email|E-Mail nicht gefunden.
action|db_query|UPDATE rex_ycom_user SET activation_key = ? WHERE email = ?|activation_key,email
action|tpl2email|resetpassword_de|email|
action|showtext|Sie erhalten eine E-Mail mit weiteren Anweisungen.|<p>|</p>|1
```

### Reset confirmation (lands user logged in, ready to change password)

```
objparams|submit_btn_show|0
objparams|send|1
objparams|csrf_protection|0
validate|ycom_auth_login|activation_key=rex_ycom_activation_key,email=rex_ycom_id|status=1|Link ungueltig oder abgelaufen.|status
action|ycom_auth_db|update
action|html|<b>Sie sind eingeloggt. Das Passwort kann nun geaendert werden.</b>
```

The `validate|ycom_auth_login` line both validates the link AND logs the user in. The next form on the same article (or the redirect target) is the password-change form.

## Terms-of-use re-acceptance

When the terms change and existing users need to re-accept:

```
objparams|form_showformafterupdate|0
ycom_auth_load_user|userinfo|email,termsofuse_accepted
hidden|termsofuse_accepted|1
php|check|label|<?php if (rex::isFrontend() && rex_ycom_auth::getUser()->getValue('termsofuse_accepted') == 1) { rex_response::sendRedirect('/'); } ?>
html|info|<p>Bitte akzeptieren Sie die neuen Nutzungsbedingungen.</p>
action|showtext|<div class="alert alert-success">Nutzungsbedingungen akzeptiert.</div>|||1
action|ycom_auth_db
```

The `php|check` line short-circuits and redirects users who have already accepted, so the form only shows for those who haven't.

## Common pitfalls

- Forgetting `csrf_protection|0` on confirmation forms reached via email links – the form rejects the request because no token exists.
- Skipping `hidden|status|1` (or whichever target status) on confirmation forms – the user stays in pending status forever.
- Using `validate|empty|password|...` AFTER `ycom_auth_password` – `ycom_auth_password` already enforces empty-checks via the rules JSON; double validation can silently fail.
- Trying to change a password without `action|ycom_auth_db` – the new password validates but never gets persisted.
- Using `action|db|rex_ycom_user` on edit forms – `db` is for inserts. Use `action|ycom_auth_db` (for the current user) or `action|manage_db|rex_ycom_user`.
- `action|copy_value|email|login` is needed when you treat email as the login – without it, `login` stays empty and `ycom_auth` validate fails.
