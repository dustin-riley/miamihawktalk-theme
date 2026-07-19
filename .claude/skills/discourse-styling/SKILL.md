---
name: discourse-styling
description: Use when editing, reviewing, or debugging any .scss file or visual styling in this Discourse theme - covers the mandatory color-variable rule, mask-based scheme-adaptive artwork, Discourse's hardcoded non-scheme-aware colors, and how to syntax-check SCSS. Also use when a style looks correct in light mode but wrong in dark mode.
---

# Discourse theme styling

Non-obvious rules for `common/`, `desktop/`, and `mobile/` SCSS in this theme.
Every item here caused a real, shipped bug. Do not re-derive them.

## ⚠️ Rule 1: Never use SCSS color variables. Always use CSS custom properties.

This is the single most important rule.

**SCSS color variables bake in the *light* scheme's values and never adapt to the
active scheme.** The theme stylesheet is not recompiled per scheme, so
`color: $primary` emits the light scheme's near-black — which made post text
invisible on the dark canvas. Same trap for `$tertiary` (links became dark
crimson on charcoal) and `$primary-low` (a light track line on a dark surface).

```scss
// WRONG — bakes the light scheme's value
color: $primary;
background: $tertiary;
border-color: $primary-low;

// RIGHT — browser resolves against the active scheme at runtime
color: var(--primary);
background: var(--tertiary);
border-color: var(--primary-low);
```

**Corollary: SCSS color functions are unusable**, since they need a real color.
Instead of `darken($tertiary, 6%)`, use Discourse's precomputed
`var(--tertiary-hover)`. Discourse ships hover/lightened variants for most
palette colors — look for an existing one rather than computing it.

The only `$` variables that are safe are the **non-color asset** ones injected
from `about.json`: `$wordmark`, `$wordmark_hawk`, `$monogram`.

**Diagnostic symptom:** an element is correct in light mode and wrong in dark
mode, while Discourse's own adjacent elements are fine → almost always a baked
SCSS color variable.

## Rule 2: Discourse hardcodes non-scheme-aware colors — override the variable

These are fixed light values in Discourse core. In dark mode they render as
light boxes, and any light text on them washes out:

| Discourse value | Exposed as | Affects |
|---|---|---|
| `$hover: #f2f2f2` | `--d-hover` | header icon buttons, notification/user-menu row hovers |
| `$selected: #d1f0ff` | `--d-selected` | autocomplete & emoji-picker selected rows |
| — | `--secondary` used as *primary-button text color* | dark text on the red button in dark mode |

**Fix by overriding the custom property, not by chasing selectors.** One
override fixes an entire family of symptoms:

```scss
:root {
  --d-hover: var(--primary-low);
  --d-selected: var(--primary-low-mid); // a step stronger, so selected ≠ hovered
}
```

Note `--d-selected-text-color` *is* scheme-aware — that mismatch (adaptive text
on a fixed light background) is exactly what makes these wash out in dark mode.

## Rule 3: Scheme-adaptive artwork requires masks, not background images

**An SVG used as `background-image` is an isolated document.** `currentColor`
and CSS custom properties cannot reach inside it, so its baked colors are fixed —
a wordmark that looks right on the light header is invisible on the dark one.

**Fix:** use the SVG as a CSS `mask` and paint it with `background-color: var(--…)`:

```scss
background-color: var(--header_primary);
mask: url($wordmark) no-repeat left center;
mask-size: contain;
```

**Masks are single-color.** The two-tone wordmark is two stacked mask layers on
the same element:
- `::before` — full wordmark, painted `var(--header_primary)`
- `::after` — "Hawk" only, painted `var(--tertiary)`

Both mask SVGs share an identical `viewBox`, text metrics, and sizing so the
overlay registers exactly. **If you resize one layer, resize both** (see
`desktop/desktop.scss`). The hawk-only SVG achieves alignment by rendering the
same full text with the other words at `fill-opacity="0"`.

Mobile swaps in the full-color monogram, so it must clear the mask
(`mask: none; background-color: transparent`) and hide the `::after` overlay.

## Rule 4: `dark-light-choose()` does not work here

It silently returns the light branch for a manually-selected dark scheme. Do not
use it to swap assets or colors. Use runtime custom properties instead.

## Rule 5: Lightened accents turn red into pink

Discourse lightens the accent for subtle UI (`--tertiary-low`, `--tertiary-400`).
That works for the default blue, but a lightened red reads pink/magenta. The
topic timeline needed explicit overrides:

```scss
.topic-timeline {
  --topic-timeline-handle-color: var(--tertiary);
  --topic-timeline-border-color: var(--primary-low);
}
```

Expect this anywhere a lightened accent is used. Relatedly, a bright accent on a
dark background drifts magenta unless its blue channel stays near its green
channel.

## Design principle: accent discipline

Red is reserved for **interactive** elements — links, primary buttons, active
nav, badges, the timeline handle. The content canvas stays neutral. Flooding
surfaces with red is a regression, not a feature.

## Finding the right variable

Read Discourse source rather than guessing selectors. Most "washed out / wrong
color" bugs resolve to a single overridable custom property:

```bash
gh api "repos/discourse/discourse/contents/<path>" --jq '.content' | base64 -d
gh api "search/code?q=repo:discourse/discourse+<term>+extension:scss" --jq '.items[].path'
```

Useful files: `app/assets/stylesheets/color_definitions.scss` (all custom
property definitions), `common/foundation/colors.scss` (the hardcoded values).

## Syntax-checking SCSS

SCSS cannot be compiled standalone — Discourse injects variables at build time.
Compile against a stub. If a file compiles with **only the asset stubs below**,
that mechanically proves no baked color variables remain:

```bash
cat > /tmp/stub.scss <<'EOF'
$wordmark:"w.svg";$wordmark_hawk:"h.svg";$monogram:"m.svg";
EOF
cat /tmp/stub.scss common/common.scss > /tmp/check.scss
npx --yes sass --no-source-map /tmp/check.scss /tmp/out.css
```

## Verifying visually

A syntax check proves nothing about correctness. **No offline tool catches a
selector that doesn't match Discourse's live DOM.** Never claim a visual fix
works without seeing it rendered — and always check **both color schemes**, since
essentially every bug in this theme has been dark-mode-only.
