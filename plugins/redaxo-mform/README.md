# redaxo-mform

Claude Code plugin for the [MForm](https://github.com/FriendsOfREDAXO/mform) REDAXO addon.

MForm is the standard form builder for REDAXO module inputs. It provides a fluent PHP API to render form fields, wrappers, repeaters and advanced widgets directly in the REDAXO backend module editor and inside YForm tables.

## Skills included

| Skill | Activates when… |
|---|---|
| `mform-basics` | Writing module input PHP, calling `MForm::factory()` and any `add*Field()` / `add*Element()` method, setting labels/attributes/options/tooltips, using fieldsets, columns, tabs, collapse, accordion, modal, inline or conditional wrappers, registering and applying templates via `MForm::registerTemplate()` / `MForm::fromTemplate()` |
| `mform-flex-repeater` | Working with `addFlexRepeaterElement()` / `addRepeaterElement()`, nested repeaters, the `__MFRID__` placeholder, supported vs. unsupported widget types inside repeaters, plain string keys for inner sub-fields |
| `mform-widgets` | Using Custom-Link (`addCustomLinkField`, `addMFormLinkField`), Media (`addMediaField`, `addMFormMediaField`), Medialist, Imagelist, Linklist or ColorSwatch widgets in modules or YForm – including all `data-*` attributes for link-type filtering, media type restrictions, ylink targeting, preview options, and `useCustomLinkForClassicWidgets()` |
| `mform-output` | Reading MForm field values in module OUTPUT PHP, calling `MFormOutputHelper::createLinkData()`, `normalizeLinkData()`, `normalizeRepeaterItems()`, `getCustomLinkUrl()`, `getCustomUrl()`, `prepareCustomLink()`, decoding repeater JSON, resolving `redaxo://` URLs |
| `mform-yform` | Using MForm-provided YForm value types (`custom_link`, `custom_link_multi`, `color_swatch`, `imagelist`, `medialist`, `linklist`) in YForm table definitions / tableset JSON, reading their stored values via YOrm datasets |

## Installation

```bash
/plugin install redaxo-mform@redaxo-marketplace
```

## Requirements

- REDAXO 5.15+
- MForm addon ≥ 9.0.0

## How the skills work together

A typical module touches several skills in sequence:

1. **`mform-basics`** – build the input form (`MForm::factory()`, `add*Field()`, wrappers, templates).
2. **`mform-widgets`** – drop in a link picker, media picker, color swatch or list widget.
3. **`mform-flex-repeater`** – wrap a section in a flex repeater for dynamic row counts.
4. **`mform-output`** – read the stored values back out in the module output.
5. **`mform-yform`** – reuse the same widgets inside a YForm table.

Each skill cross-references the others where relevant, so Claude pulls in the right one automatically based on what file / context you are editing.
