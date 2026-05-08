---
name: rex-create-module
description: Scaffold a new REDAXO module with input/output PHP, REX_VALUE placeholders, and proper escaping. Asks for a name and the fields it should contain. Invoke as /redaxo-core:rex-create-module when the user wants a new module scaffolded.
---

You are helping the user scaffold a new REDAXO module. A module has two parts:

- **Input** – the editor form shown in the backend (uses `name="REX_INPUT_VALUE[N]"`)
- **Output** – the frontend rendering (uses `'REX_VALUE[N]'`, always with `rex_escape()`)

## Steps

1. **Ask the user** for:
   - The module name (human-readable, e.g. "Hero Banner")
   - A list of fields. For each field, the user gives a label and a type. Supported types:
     - `text` – single-line text
     - `textarea` – multi-line text
     - `wysiwyg` – rich text (escape with `'html_simplified'` on output)
     - `media` – single image/file (uses `REX_MEDIA[N]` widget)
     - `medialist` – multiple media (uses `REX_MEDIALIST[N]`)
     - `link` – single internal link (uses `REX_LINK[N]`)
     - `linklist` – multiple internal links (uses `REX_LINKLIST[N]`)
   - The output goal – plain HTML, a Bootstrap component, a Tailwind structure, etc.

2. **Verify limits**: at most 20 `REX_VALUE` slots, 10 each of `REX_MEDIA`, `REX_MEDIALIST`, `REX_LINK`, `REX_LINKLIST`. If the user wants more, suggest storing JSON in one `REX_VALUE` slot or recommending the `yform` addon.

3. **Generate two files**:
   - `<name>_input.php` – the editor form, using `class="form-control"` Bootstrap-style markup that REDAXO ships with in the backend.
   - `<name>_output.php` – the frontend rendering. **Every value touched by an editor must go through `rex_escape()`** with the right strategy.

4. **Wrap the input in a `<fieldset>`** with a `<legend>` matching the module name, so the editor view stays organized.

5. **For media output**, look up the `rex_media` instance to get dimensions and alt text:

```php
<?php if ($filename = 'REX_MEDIA[1]') { $media = rex_media::get($filename); } ?>
<?php if ($media): ?>
    <img src="<?= rex_url::media($media->getFileName()) ?>"
         alt="<?= rex_escape($media->getTitle()) ?>"
         width="<?= $media->getWidth() ?>"
         height="<?= $media->getHeight() ?>">
<?php endif; ?>
```

6. **For link output**, look up the `rex_article`:

```php
<?php $articleId = (int) 'REX_LINK[1]'; $article = $articleId ? rex_article::get($articleId) : null; ?>
<?php if ($article): ?>
    <a href="<?= $article->getUrl() ?>"><?= rex_escape($article->getName()) ?></a>
<?php endif; ?>
```

7. **Output the two files** as separate code blocks the user can copy into the REDAXO backend (Modules → Add new module). Add a short comment at the top of each block: `<?php // Input` and `<?php // Output`.

8. **Don't write to disk** – modules are stored in the database, not as files. The user pastes the code into the backend UI.

After generating, briefly explain what you produced and remind them to clear the REDAXO cache after creating the module if it's already in use.
