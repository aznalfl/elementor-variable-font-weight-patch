# Elementor Variable Font Weight Patch

A temporary WordPress compatibility patch that fixes missing font-weight
CSS on Elementor Global Typography presets built on registered
variable-weight fonts (e.g. [Geist](https://vercel.com/font)).

## The problem

Elementor's **Global Fonts** (Site Settings → Global Fonts) let you assign
a font-weight to each typography preset. When the assigned font is a
registered *variable-weight* font, Elementor adds a second "weight"
slider control for the variable font-weight axis - but due to a bug in
Elementor core, the resulting
`--e-global-typography-{id}-font-weight` CSS custom property (referenced
by widgets on the front end) is never actually defined anywhere.

The configured weight is saved correctly in the database and Elementor
*references* the variable correctly - it just never *defines* it, so
affected text silently falls back to the browser default weight.

This is tracked upstream as
[elementor/elementor#29496](https://github.com/elementor/elementor/issues/29496)
and
[elementor/elementor#36009](https://github.com/elementor/elementor/issues/36009).

## What this plugin does

It does **not** touch Elementor core. At render time, it reads the
already-saved, correct values from the active Elementor Site Kit and
prints the missing CSS custom properties so the browser can resolve them.

Specifically, on eligible front-end requests it:

1. Fetches the active Elementor Kit for the current request.
2. Reads `system_typography` and `custom_typography` settings and
   collects every preset with a `typography_weight` value (Elementor only
   populates that field for presets using a registered variable-weight
   font, so only affected presets are ever touched - no font list or
   preset-name matching required).
3. Sanitises each preset id and weight value.
4. Emits a compact `:root { ... }` block declaring both
   `--e-global-typography-{id}-font-weight` (the name Elementor's widget
   CSS actually references) and `--e-global-typography-{id}-weight` (seen
   in some existing manual workarounds), for drop-in compatibility with
   either naming convention.
5. Attaches that CSS as an inline style on the `elementor-frontend`
   stylesheet, falling back to a `<style>` tag in `<head>` if that handle
   isn't registered.

The plugin only runs on standard front-end page views - it skips admin,
AJAX, REST, cron, WP-CLI, and the login screen.

## Requirements

- WordPress 6.0+
- PHP 7.4+
- Elementor (any version affected by the linked issues)

## Installation

1. Download or clone this repository.
2. Upload the `elementor-variable-font-weight-patch` folder to
   `wp-content/plugins/`, or zip it and upload via
   **Plugins → Add New → Upload Plugin**.
3. Activate **Elementor Variable Font Weight Patch** from the Plugins
   screen.

No configuration is required - it works automatically once activated.

## Disabling without deactivating

To turn off all output from this plugin without deactivating it (e.g. for
debugging), define the following in `wp-config.php` (or anywhere loaded
before the `plugins_loaded` hook):

```php
define( 'ELEMENTOR_VARIABLE_FONT_WEIGHT_PATCH_DISABLE', true );
```

## Is it safe to keep active after Elementor fixes this upstream?

Yes. If Elementor fixes this by sourcing the same "weight" slider value
this plugin reads, the duplicate CSS custom property declaration will
resolve to the same value - CSS custom properties are idempotent when
redeclared identically, so there's no functional conflict.

The one edge case: if a preset has a stale/unused legacy `font_weight`
value left over from before it was switched to a variable font, and
Elementor's fix surfaces that value instead, the two declarations could
differ for that preset specifically. The later declaration in the cascade
wins - this is a display difference, not an error.

## Removal

Once the linked Elementor issues are resolved in a released version, this
plugin is no longer needed. Simply deactivate and delete it from the
Plugins screen - there is nothing else to clean up.

## License

GPL-2.0-or-later
