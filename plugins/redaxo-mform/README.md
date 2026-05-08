# redaxo-mform

Claude Code plugin for the [MForm](https://github.com/FriendsOfREDAXO/mform) REDAXO addon.

MForm is the standard form builder for REDAXO module inputs. It provides a fluent PHP API to render form fields, wrappers, repeaters and advanced widgets directly in the REDAXO backend module editor.

## Skills included

| Skill | Activates when… |
|---|---|
| `mform-basics` | Writing module input PHP, using `MForm::factory()`, calling `add*Field()` methods, setting labels/attributes/options |
| `mform-flex-repeater` | Working with `addFlexRepeaterElement()` / `addRepeaterElement()`, nested repeaters, or any dynamic list in a module |
| `mform-widgets` | Using Custom-Link, Imagelist, Medialist, Linklist or ColorSwatch widgets in modules or YForm |
| `mform-output` | Reading repeater/module values in templates, calling `MFormOutputHelper::createLinkData()` or `normalizeLinkData()` |
| `mform-yform` | Using MForm-provided YForm value types (`custom_link`, `color_swatch`, `imagelist` etc.) in YForm table definitions |

## Installation

```bash
/plugin install redaxo-mform@redaxo-marketplace
```

## Requirements

- REDAXO 5.15+
- MForm addon ≥ 9.0.0
