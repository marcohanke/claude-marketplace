---
name: ycom-quickstart
description: One-shot recipe to bootstrap a complete YCom community frontend on a REDAXO instance that has the FriendsOfREDAXO `api` addon installed. Creates the 9 articles (login, logout, register + confirm, password forgot + reset, profile, password change, terms of use), wires their YForm pipe code into slices on the standard "YForm Formbuilder" module, sets per-article ycom_auth_type, inserts the email templates, and writes the ycom/auth config keys. Use when the user asks to "set up YCom community pages", "scaffold the YCom frontend", "create the login/registration flow", or runs a fresh REDAXO instance that needs YCom wired up programmatically. Skip when the instance has no `api` addon (offer the manual setup from `ycom-forms` instead).
---

# YCom Community Quickstart

Bootstraps a working YCom frontend (registration → email confirmation → login → profile → password reset → terms → logout) on a REDAXO instance in one pass. Idempotent. Recipe is **dogfood-validated** end-to-end against YCom 4.4.4 + YForm 4.x + REDAXO 5.17 + the FriendsOfREDAXO `api` addon — every curl call and every pipe snippet below has been run against a real instance.

## Why this is hybrid (api addon + PHP companion), not pure-API and not browser

The `api` addon doesn't reach everything we need. The matrix:

| Task | api addon | This skill uses |
|---|---|---|
| Category, Article, Template lookup | ✅ | api |
| Slice with module + REX_VALUE values | ✅ | api |
| `ycom_auth_type` column on `rex_article` | ❌ | PHP companion (SQL) |
| `rex_yform_email_template` row | ❌ | PHP companion |
| `rex_config` namespace `ycom/auth` | ❌ | PHP companion (`setConfig()`) |

Browser automation (Chrome / Playwright) for the setup itself is the wrong tool: fragile, slow, no idempotency. Browser is fine for the **optional E2E smoke test at the end** — and there Playwright beats Chrome (fresh context, no admin-session bleed into frontend permission checks).

## Pre-flight

```bash
TOKEN="..."                 # API bearer token from "AddOns → API → Token"
BASE="https://example.local"

# All four must be available
ssh user@host 'cd /path/to/redaxo && bin/console package:list' \
  | grep -E '(ycom|yform|yrewrite|api)'

# Token works against HTTPS — Apache often redirects HTTP→HTTPS and strips
# the Authorization header on the redirect, so always call HTTPS directly.
curl -sk -H "Authorization: Bearer $TOKEN" "$BASE/api/templates" | head -c 80
# Expect a JSON {"data":[...]} — not {"error":"Authorization failed"}.
```

If you get "Authorization failed" on HTTPS too, check `.htaccess`:
```apache
RewriteCond %{HTTP:Authorization} .
RewriteRule ^ - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

## Step 1 — Create category and 9 articles via the api addon

```bash
PARENT=0
TEMPLATE=1
CLANG=1                     # de — adjust if your active language differs

# 1a) Category. NOTE: no trailing slash on the path. The parent is named
# `category_id` (NOT parent_id). Response shape: {"message":..., "id":N}.
COMMUNITY=$(curl -sk -X POST "$BASE/api/structure/categories" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"name\":\"Community\",\"category_id\":$PARENT,\"status\":1}" \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["id"])')

# 1b) Articles — one per page. Helper:
mkart() {
  curl -sk -X POST "$BASE/api/structure/articles" \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d "{\"name\":\"$1\",\"category_id\":$COMMUNITY,\"template_id\":$TEMPLATE,\"status\":1}" \
    | python3 -c 'import sys,json; print(json.load(sys.stdin)["id"])'
}

LOGIN=$(mkart "Login")
LOGOUT=$(mkart "Logout")
REGISTER=$(mkart "Registrierung")
REGISTER_CONFIRM=$(mkart "Registrierung bestaetigen")
FORGOT=$(mkart "Passwort vergessen")
RESET=$(mkart "Passwort zuruecksetzen")
PROFILE=$(mkart "Profil")
PWCHANGE=$(mkart "Passwort aendern")
TERMS=$(mkart "Nutzungsbedingungen akzeptieren")

echo "$LOGIN $LOGOUT $REGISTER $REGISTER_CONFIRM $FORGOT $RESET $PROFILE $PWCHANGE $TERMS"
```

Avoid umlauts in article names — the slug-generator will UTF-8-escape them and you'll fight URL parsing later. Use `bestaetigen` not `bestätigen`, `aendern` not `ändern`. The form labels can still use proper umlauts.

The api addon doesn't accept custom columns (`ycom_auth_type`) at create-time; we set those in step 3.

## Step 2 — Slices with the YForm Formbuilder module

The standard "YForm Formbuilder" module (key `yform`) reads the **pipe code from REX_VALUE[3]**. REX_VALUE[1] is the "on submit" preset (empty = "let the form's own actions decide" — that's what we want, since YCom forms call `action|ycom_auth_db`, `action|tpl2email` inline).

Look up the module id by key (don't hardcode — varies per instance):

```bash
MODULE=$(curl -sk -H "Authorization: Bearer $TOKEN" "$BASE/api/modules?per_page=200" \
  | python3 -c "import sys,json; print(next(m['id'] for m in json.load(sys.stdin)['data'] if m.get('key')=='yform'))")
```

If `key='yform'` doesn't exist on the instance, the demo modules weren't installed. You can create one via `POST /api/modules`, but the simpler workaround is to install the `developer` addon and pull the standard YForm module from there.

Helper for slicing — pipe code goes into the JSON body cleanly via Python:

```bash
mkslice() {
  local article_id="$1"
  local pipe_file="$2"
  local payload=$(python3 -c "import json; print(json.dumps({
    'module_id': $MODULE,
    'clang_id': $CLANG,
    'ctype_id': 1,
    'value3': open('$pipe_file').read()
  }))")
  curl -sk -X POST "$BASE/api/structure/articles/$article_id/slices" \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    -d "$payload"
  echo
}
```

Slice-create response: `{"message":"ArticleSlice created","slice_id":N}` — note `slice_id`, not `id`.

### Pipe codes — write to files first, paste content unchanged

These are the canonical pipe codes from the YCom docs (`docs/03_login_logout_profile_register.md`, `05_passwords.md`, `11_tokens.md` — token-based, NOT the older `activation_key` style). Save each as a separate file under `/tmp/ycom-pipes/` for predictable shell quoting.

**login.pipe**
```
validate|ycom_auth|login|password|stayfield|Bitte Login und Passwort eingeben|Login fehlgeschlagen
text|login|Benutzername|||{"autocomplete":"username"}
password|password|Passwort|||{"autocomplete":"current-password"}
checkbox|stayfield|Eingeloggt bleiben
ycom_auth_returnto|returnTo|
```

**logout.pipe**
```
ycom_auth_logout|Logout|
```

**register.pipe**
```
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
ycom_user_token|token|create|register|email
action|tpl2email|register_de_template|email|
action|showtext|<div class="alert alert-success">Bitte pruefen Sie Ihre E-Mails und klicken Sie den Bestaetigungslink.</div>|||1
```

**register_confirm.pipe** — note: NO `objparams|send|1`, NO `objparams|submit_btn_show|0`. The user must click a button. See pitfalls below.
```
ycom_user_token|token|validate|register|Token ist ungueltig oder abgelaufen.
hidden|status|1
objparams|csrf_protection|0
action|ycom_auth_db|update
action|html|<div class="alert alert-success">Vielen Dank, Ihre E-Mail wurde bestaetigt. Sie sind nun eingeloggt.</div>
```

**forgot.pipe**
```
text|email|E-Mail
validate|type|email|email|Bitte gueltige E-Mail eingeben.
validate|empty|email|Bitte E-Mail eingeben.
validate|in_table|email|rex_ycom_user|email|E-Mail nicht gefunden.
ycom_user_token|token|create|password_reset|email
action|tpl2email|resetpassword_de_template|email|
action|showtext|Sie erhalten eine E-Mail mit weiteren Anweisungen.|<p>|</p>|1
```

**reset.pipe** — same no-auto-submit rule as register_confirm.
```
ycom_user_token|token|validate|password_reset|Link ungueltig oder abgelaufen.
hidden|new_password_required|1
objparams|csrf_protection|0
action|ycom_auth_db|update
action|html|<div class="alert alert-success">Sie sind eingeloggt. Bitte setzen Sie nun ein neues Passwort.</div>
```

**profile.pipe**
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

**pwchange.pipe**
```
password|old_password|Bisheriges Passwort||no_db|{"autocomplete":"current-password"}
validate|empty|old_password|Bitte bisheriges Passwort angeben.
validate|ycom_auth_password|old_password|Bisheriges Passwort ist nicht korrekt.
ycom_auth_password|password|Neues Passwort*|{"length":{"min":10},"letter":{"min":1},"digit":{"min":1}}|Passwort-Anforderungen nicht erfuellt.
password|password_2|Passwort wiederholen||no_db
validate|compare|password|password_2|!=|Passwoerter stimmen nicht ueberein.
action|showtext|<div class="alert alert-success">Passwort wurde aktualisiert.</div>|||1
action|ycom_auth_db
hidden|new_password_required|0
```

**terms.pipe**
```
objparams|form_showformafterupdate|0
ycom_auth_load_user|userinfo|email,termsofuse_accepted
hidden|termsofuse_accepted|1
php|check|label|<?php if (rex::isFrontend() && rex_ycom_auth::getUser() && rex_ycom_auth::getUser()->getValue('termsofuse_accepted') == 1) { rex_response::sendRedirect('/'); } ?>
html|info|<p>Bitte akzeptieren Sie die neuen Nutzungsbedingungen.</p>
checkbox|termsofuse_accepted_check|Ich akzeptiere die Nutzungsbedingungen.||no_db
validate|empty|termsofuse_accepted_check|Bitte akzeptieren Sie die Nutzungsbedingungen.
action|showtext|<div class="alert alert-success">Nutzungsbedingungen akzeptiert.</div>|||1
action|ycom_auth_db
```

Then push them all:
```bash
mkslice $LOGIN              /tmp/ycom-pipes/login.pipe
mkslice $LOGOUT             /tmp/ycom-pipes/logout.pipe
mkslice $REGISTER           /tmp/ycom-pipes/register.pipe
mkslice $REGISTER_CONFIRM   /tmp/ycom-pipes/register_confirm.pipe
mkslice $FORGOT             /tmp/ycom-pipes/forgot.pipe
mkslice $RESET              /tmp/ycom-pipes/reset.pipe
mkslice $PROFILE            /tmp/ycom-pipes/profile.pipe
mkslice $PWCHANGE           /tmp/ycom-pipes/pwchange.pipe
mkslice $TERMS              /tmp/ycom-pipes/terms.pipe
```

If you need to fix a slice afterwards:
```bash
# Endpoint is nested under article — NOT /api/structure/slices/{id}.
# Both PUT and PATCH work; both return {"message":"Slice updated","slice_id":N}.
curl -sk -X PATCH "$BASE/api/structure/articles/$ARTICLE_ID/slices/$SLICE_ID" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"value3": "..."}'
```

## Step 3 — PHP companion: permissions, email templates, ycom config

Save as `/tmp/ycom-quickstart.php`. Run after step 2 from anywhere; it autodetects the REDAXO root by walking up from CWD looking for `redaxo/src/core/boot.php` (or set `REDAXO_ROOT=...` explicitly).

```php
<?php
// /tmp/ycom-quickstart.php — YCom Quickstart Companion.
// Idempotent. Re-run safely.

$rexRoot = $_SERVER['REDAXO_ROOT'] ?? null;
if (!$rexRoot) {
    $cwd = getcwd();
    while ($cwd !== '/' && !file_exists($cwd . '/redaxo/src/core/boot.php')) {
        $cwd = dirname($cwd);
    }
    if ($cwd === '/') {
        fwrite(STDERR, "Cannot find REDAXO root. Set REDAXO_ROOT=/path/to/redaxo.\n");
        exit(1);
    }
    $rexRoot = $cwd;
}

$REX = ['REDAXO' => true, 'HTDOCS_PATH' => $rexRoot . '/', 'BACKEND_FOLDER' => 'redaxo', 'LOAD_PAGE' => false];
require $rexRoot . '/redaxo/src/core/boot.php';
require $rexRoot . '/redaxo/src/core/packages.php';

$ids = [];
foreach (array_slice($_SERVER['argv'], 1) as $arg) {
    if (preg_match('/^([A-Z_]+)=(\d+)$/', $arg, $m)) {
        $ids[$m[1]] = (int) $m[2];
    }
}
foreach (['LOGIN','LOGOUT','REGISTER','REGISTER_CONFIRM','FORGOT','RESET','PROFILE','PWCHANGE','TERMS'] as $key) {
    if (empty($ids[$key])) { fwrite(STDERR, "Missing $key=<id>\n"); exit(1); }
}

// 3a) ycom_auth_type per article. UPDATE will hit ALL clangs of the article —
// that's the desired default. Add WHERE clang_id = ... if you want per-language perms.
$perms = [
    $ids['LOGIN']            => 2, // only NOT logged in
    $ids['LOGOUT']           => 1, // only logged in
    $ids['REGISTER']         => 2,
    $ids['REGISTER_CONFIRM'] => 0, // open via email link
    $ids['FORGOT']           => 0,
    $ids['RESET']            => 0,
    $ids['PROFILE']          => 1,
    $ids['PWCHANGE']         => 1,
    $ids['TERMS']            => 1,
];
$sql = rex_sql::factory();
foreach ($perms as $articleId => $perm) {
    $sql->setQuery(
        'UPDATE ' . rex::getTable('article') . ' SET ycom_auth_type = ? WHERE id = ?',
        [$perm, $articleId]
    );
}

// 3b) Email templates. The unique key is `name` (no `key` column — verified
// against YForm 4.x schema: id, name UNIQUE, mail_from, mail_from_name,
// mail_reply_to, mail_reply_to_name, subject, body, body_html, attachments, updatedate).
$templates = [
    'register_de_template' => [
        'subject' => 'Bitte bestaetigen Sie Ihre Registrierung',
        'body'    => "Hallo,\n\nbitte klicken Sie diesen Link, um Ihre Registrierung zu bestaetigen:\n\n"
                  .  "<?= trim(rex::getServer(), '/') . rex_getUrl({$ids['REGISTER_CONFIRM']}, null, ['token' => 'REX_YFORM_DATA[field=token]']) ?>\n\n"
                  .  "Wenn Sie sich nicht registriert haben, ignorieren Sie diese E-Mail.\n",
    ],
    'resetpassword_de_template' => [
        'subject' => 'Passwort zuruecksetzen',
        'body'    => "Hallo,\n\nklicken Sie diesen Link, um Ihr Passwort zurueckzusetzen:\n\n"
                  .  "<?= trim(rex::getServer(), '/') . rex_getUrl({$ids['RESET']}, null, ['token' => 'REX_YFORM_DATA[field=token]']) ?>\n\n"
                  .  "Wenn Sie keine Anfrage gestellt haben, ignorieren Sie diese E-Mail.\n",
    ],
];
foreach ($templates as $name => $tpl) {
    $sql->setQuery(
        'INSERT INTO ' . rex::getTable('yform_email_template')
        . ' (name, mail_from, mail_from_name, mail_reply_to, mail_reply_to_name, subject, body, body_html, attachments, updatedate) '
        . ' VALUES (?, ?, ?, "", "", ?, ?, "", "", NOW()) '
        . ' ON DUPLICATE KEY UPDATE subject = VALUES(subject), body = VALUES(body), updatedate = NOW()',
        [$name, rex::getErrorEmail(), rex::getServerName(), $tpl['subject'], $tpl['body']]
    );
}

// 3c) ycom/auth config keys
$plugin = rex_plugin::get('ycom', 'auth');
$cfg = [
    'article_id_login'           => $ids['LOGIN'],
    'article_id_logout'          => $ids['LOGOUT'],
    'article_id_register'        => $ids['REGISTER'],
    'article_id_password'        => $ids['FORGOT'],
    'article_id_jump_ok'         => $ids['PROFILE'],
    'article_id_jump_logout'     => $ids['LOGIN'],
    'article_id_jump_denied'     => $ids['LOGIN'],
    'article_id_jump_password'   => $ids['PWCHANGE'],
    'article_id_jump_termsofuse' => $ids['TERMS'],
];
foreach ($cfg as $key => $value) {
    $plugin->setConfig($key, (int) $value);
}

// 3d) Refresh caches
rex_yform_manager_table::deleteCache();
rex_delete_cache();
if (class_exists('rex_yrewrite')) {
    rex_yrewrite::generatePathFile([]);
}
echo "Done.\n";
```

Run:
```bash
php /tmp/ycom-quickstart.php \
  LOGIN=$LOGIN LOGOUT=$LOGOUT REGISTER=$REGISTER REGISTER_CONFIRM=$REGISTER_CONFIRM \
  FORGOT=$FORGOT RESET=$RESET PROFILE=$PROFILE PWCHANGE=$PWCHANGE TERMS=$TERMS
```

## Step 4 — Smoke-test (optional)

If your dev instance has mail capture (MailHog at `localhost:1025`, Mailpit, or read `rex_yform_email_template` send log), verify a real flow. The token from the email URL:

```
http(s)://example.local/de/community/registrierung-bestaetigen/?token=<verifier20chars><base64hash>
```

The `==` base64 padding gets URL-encoded to `%3D%3D`. When pasting into a curl test, leave it encoded.

Manual flow:
1. `/registrierung` → fill + submit → "Bitte pruefen Sie Ihre E-Mails…"
2. Pull link from MailHog → click → "Vielen Dank, Ihre E-Mail wurde bestaetigt." → user is logged in (status flips 0 → 1)
3. `/profil` shows your name
4. `/community/logout/` → redirects to `/login/`
5. Login with credentials → redirects to `/profil/`

For automation, write a Playwright spec — Chrome via MCP would inherit your admin session and confuse the auth checks; Playwright's fresh context avoids that.

## Idempotency

Steps 2 and 3 are idempotent **as long as the article IDs don't change**. Save them:

```bash
cat > .ycom-quickstart-state.json <<EOF
{"login":$LOGIN,"logout":$LOGOUT,"register":$REGISTER,"register_confirm":$REGISTER_CONFIRM,"forgot":$FORGOT,"reset":$RESET,"profile":$PROFILE,"pwchange":$PWCHANGE,"terms":$TERMS}
EOF
```

Re-run guard for step 1: query `GET /api/structure/articles?per_page=200` and skip names that already exist under the Community category. Step 2 (slices): query `GET /api/structure/articles/$ID/slices` and PATCH the existing slice instead of POSTing a new one. Step 3 is fully idempotent (UPDATE, INSERT…ON DUPLICATE KEY UPDATE, setConfig).

## Common pitfalls (all observed during dogfood)

- **Trailing slash on `categories`**: `POST /api/structure/categories` works, `POST /api/structure/categories/` returns `{"error":"Route not found"}`. The `/articles` endpoint accepts both.
- **Field `category_id`, not `parent_id`**: when creating a category, the *parent* is `category_id` (semi-counterintuitive). For articles, `category_id` is the containing category.
- **Response shape**: `POST /articles` returns `{"message":"Article created","id":N}` — there is no `data` wrapper. Slice POST returns `slice_id`, not `id`.
- **Slice update endpoint is nested**: `PATCH /api/structure/articles/{art}/slices/{slice}`. Both `PATCH /api/structure/slices/{slice}` and `PATCH /api/slices/{slice}` are 404.
- **HTTP→HTTPS redirect strips `Authorization`**: always call `https://…` directly. With self-signed dev certs add `-k` to curl.
- **Authorization header missing under Apache**: needs the `RewriteRule ^ - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]` snippet in `.htaccess`. Symptom is 401 even with a valid token.
- **`clang_id` is mandatory on slice POST** when the instance has more than one language. Skipping it silently slices into the default clang only.
- **`rex_yform_email_template` schema has no `key` column**. Insert into `name` (UNIQUE) instead. Other columns: `mail_from`, `mail_from_name`, `mail_reply_to`, `mail_reply_to_name`, `subject`, `body`, `body_html`, `attachments`, `updatedate`.
- **Avoid umlauts in article names**: `bestaetigen` not `bestätigen`. The yrewrite slug-generator handles them, but downstream tooling that copy-pastes URLs gets messy.
- **The "YForm Formbuilder" module reads pipe code from REX_VALUE[3]**, NOT REX_VALUE[1]. REX_VALUE[1] is the "on submit" preset — leave it empty so the YCom-specific `action|ycom_auth_db` and `action|tpl2email` from your pipe take effect.
- **DO NOT add `objparams|send|1` to token-validation pages** (register-confirm, password-reset). The `ycom_user_token` value field reads the token from the URL into the form value only when `$params['send'] !== true`. With `send=1` the validator runs against an empty value and you get "Token ist ungueltig". The original docs (`11_tokens.md`) intentionally use a regular submit button — keep it that way.
- **YCom hardcodes the post-confirm redirect to `article_id_login`**: after `ycom_user_token|validate` succeeds, the user is redirected to the login article (not `article_id_jump_ok`). On a perm-2 login page this means an immediately-logged-in user gets bounced through `article_id_jump_denied`. Setting `article_id_jump_denied = LOGIN` produces a stable end state; alternatively set `article_id_jump_denied = PROFILE` if you'd rather land them there.
- **The fourth pipe slot of `ycom_user_token|<name>|create|<type>|<email_field>` is documentation-only** — the implementation always reads from a form field literally named `email`. So the form must contain a `text|email|…` field.
- **`rex::getServer()` may return HTTP** even on HSTS hosts — emails will contain `http://` links that the browser auto-upgrades to HTTPS. Cosmetic, not broken.
- **PHP deprecation noise on token validate**: `substr(null, ...)` in `ycom_user_token.php:23-24` shows up when `debug.enabled = true` and the token field rendered before the value was loaded. Cosmetic.

## Out of scope (use these other skills)

- **OTP / 2FA** → `ycom-otp`. Add an extra article + `ycom_auth_otp|setup` pipe and set `otp_article_id` via `setConfig()`.
- **SAML / OAuth2 / CAS** → `ycom-external-auth`.
- **Group-scoped article permissions** → `ycom-permissions`. Activate the `group` plugin first, then set `ycom_group_type` and `ycom_groups` per article.
- **Second language (en, fr, …)** → duplicate articles via the api addon's article-duplicate endpoint, translate slices, create per-language email templates (`register_en_template`, …). Keep `ycom/auth` config keys language-agnostic — they're shared across clangs.
- **Bootstrap/Tailwind styling** → ship custom `ytemplates/` from your own addon; the pipe codes here render plain HTML.
