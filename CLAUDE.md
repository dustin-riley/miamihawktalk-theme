# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **full Discourse theme** (not a theme component) for miamihawktalk.fans, a Miami
University RedHawks fan forum. It is distributed as a git repository: Discourse
pulls it directly from GitHub. There is no build step, no package manager, no
test runner, and no server access involved — the deliverable is the source files
themselves.

`component: false` in `about.json` is what makes it a full theme, which is why it
can own the color schemes and header.

## Deploy / iteration loop

Changes reach the live site only through git:

```bash
git push origin main
```

Then in Discourse: **Admin → Customize → Themes → MiamiHawkTalk → Update**, and
**hard-refresh** the browser (Cmd+Shift+R). Both steps matter — the theme only
recompiles when it actually pulls, and stale CSS is a common false "my change
didn't work."

`main` is the deploy branch; Discourse tracks the repo's default branch.

Optional live-reload during development:

```bash
gem install discourse_theme && discourse_theme watch .
```

## Verifying changes

There are no unit tests. Verification is (1) offline structural checks and
(2) looking at the running site. **Only the site can confirm CSS actually
matches Discourse's live DOM** — no offline tool catches a selector mismatch, so
never claim a visual fix works without seeing it rendered in both color schemes.

Offline checks:

```bash
python3 -m json.tool about.json > /dev/null      # manifest
ruby -ryaml -e "YAML.load_file('locales/en.yml')" # locales
xmllint --noout assets/*.svg                      # artwork
```

SCSS cannot be compiled standalone — Discourse injects color variables and
assets at build time. To syntax-check, compile against a stub of those variables:

```bash
cat > /tmp/stub.scss <<'EOF'
$primary:#000;$secondary:#fff;$tertiary:#A6192E;$quaternary:#A6192E;
$primary-low:#e9e9e9;$primary-high:#333;$header_background:#fff;$header_primary:#000;
$highlight:#eee;$danger:#A6192E;$success:#1E7A46;$love:#A6192E;
$wordmark:"w.svg";$wordmark_hawk:"h.svg";$monogram:"m.svg";
EOF
cat /tmp/stub.scss common/common.scss > /tmp/check.scss
npx --yes sass --no-source-map /tmp/check.scss /tmp/out.css
```

Regenerate the favicon from the monogram after changing it:

```bash
rsvg-convert -w 512 -h 512 assets/monogram.svg -o assets/favicon.png
```

## Architecture

**Color schemes drive everything.** `about.json` defines two schemes
(`MiamiHawkTalk Light`, `MiamiHawkTalk Dark`). Discourse exposes each scheme's
colors both as SCSS variables (`$primary`, `$tertiary`, `$primary-low`, …) and as
CSS custom properties (`var(--primary)`, `var(--tertiary)`, `var(--primary-low)`, …).

> ### ⚠️ Never use SCSS color variables. Always use CSS custom properties.
>
> This is the single most important rule in this repo. **SCSS color variables
> bake in the *light* scheme's values and never adapt to the active scheme.**
> The theme stylesheet is not recompiled per scheme, so `color: $primary`
> emits the light scheme's near-black — which rendered post text invisible on
> the dark canvas until it was fixed. Same trap for `$tertiary` (links became
> dark crimson on charcoal) and `$primary-low`.
>
> Write `color: var(--primary)`, `background: var(--tertiary)`,
> `border-color: var(--primary-low)`. The browser resolves these at runtime
> against whichever scheme the user selected.
>
> Corollary: SCSS color functions can't be used, since they need a real color.
> Instead of `darken($tertiary, 6%)`, use Discourse's precomputed
> `var(--tertiary-hover)`. Discourse ships hover/lightened variants for most
> palette colors — look for one rather than computing it.
>
> The `$` variables that *are* safe are the non-color asset ones injected from
> `about.json` (`$wordmark`, `$wordmark_hawk`, `$monogram`).
>
> **Symptom to watch for:** an element is correct in light mode and wrong in
> dark mode, while Discourse's own adjacent elements are fine. That is almost
> always a baked SCSS color variable.

**Assets** declared in `about.json`'s `assets` block become SCSS variables of the
same name (`wordmark` → `$wordmark`), used as `url($wordmark)`.

**Styling files:** `common/` applies everywhere; `desktop/` and `mobile/` refine
it. Mobile swaps the wordmark for the monogram.

**Design principle — accent discipline:** red is reserved for interactive
elements (links, primary buttons, active nav, badges, the timeline handle). The
content canvas stays neutral. Preserve this when adding styles; flooding surfaces
with red is a regression, not a feature.

## Discourse gotchas (learned the hard way — do not re-derive)

These caused real bugs in this theme. They are non-obvious and repeat often:

1. **An SVG used as `background-image` is an isolated document.** `currentColor`
   and CSS custom properties cannot reach inside it, so its baked colors are
   fixed. A wordmark that looks right on the light header is invisible on the
   dark one. **Fix:** use the SVG as a CSS `mask` and paint it with
   `background-color: var(--…)`, which the browser resolves per active scheme.

2. **`dark-light-choose()` does not reliably resolve for a manually-selected
   dark scheme.** It silently returns the light branch. Do not use it to swap
   assets or colors; use runtime CSS custom properties instead.

3. **Masks are single-color.** The two-tone wordmark is built from two stacked
   mask layers on the same element (`::before` = full wordmark painted
   `var(--header_primary)`; `::after` = "Hawk" only, painted `var(--tertiary)`).
   Both mask SVGs share the identical `viewBox`, text metrics, and sizing so the
   overlay registers exactly. **If you resize one layer, resize both** — see
   `desktop/desktop.scss`.

4. **Discourse hardcodes several non-scheme-aware colors**, which render as
   light boxes in dark mode and wash out light text:
   - `$hover: #f2f2f2` → exposed as `--d-hover`, used by header icon buttons and
     notification/user-menu row hovers.
   - `$selected: #d1f0ff` → exposed as `--d-selected`, used by autocomplete /
     emoji-picker selected rows. Its text color (`--d-selected-text-color`) *is*
     scheme-aware, so light text landed on a light-blue box. Fixed the same way.
   - `--secondary` is used as *primary-button text color*, so it goes dark in the
     dark scheme (dark text on a red button).

   **Fix these by overriding the CSS custom property** (e.g.
   `:root { --d-hover: var(--primary-low); }`), not by chasing individual
   selectors. One variable override fixes an entire family of symptoms.

5. **Derived tints turn red into pink.** Discourse lightens the accent for
   subtle UI (`--tertiary-low`, `--tertiary-400`) — fine for the default blue,
   but a lightened red reads as pink/magenta. The topic timeline needed explicit
   `--topic-timeline-handle-color` / `--topic-timeline-border-color` overrides.
   Expect this wherever a lightened accent is used.

6. **A bright accent on a dark background can read magenta.** Keep the dark
   accent's blue channel near its green channel or it drifts pink.

7. **The favicon is a Discourse site setting, not a theme file.** `assets/favicon.png`
   is produced here but must be uploaded manually under
   **Admin → Settings → Branding**. The theme cannot set it.

8. **Find the right variable by reading Discourse source** rather than guessing
   selectors — `gh api repos/discourse/discourse/contents/<path>` works well.
   Most "washed out / wrong color" bugs resolve to one overridable custom
   property.

## Branding constraints

All artwork in `assets/` is **original**. Miami University's actual trademarks
(the RedHawk logo, beveled "M", official wordmarks) are deliberately **not** used
— this is an unofficial fan site. Do not introduce university marks.

## Project docs

- `docs/ROADMAP.md` — feature backlog and phasing. Phase 1 (branded theme) is
  complete; Phase 2 (custom homepage, Glimmer components) and Phase 3 (features
  needing live sports data) are planned. **Phase 1 is deliberately near-zero
  JavaScript** — no `.gjs`, no `api.renderInOutlet` — which is why it is robust
  across Discourse upgrades. Adding JS moves work into Phase 2 scope.
- `docs/superpowers/specs/` and `docs/superpowers/plans/` — the Phase 1 design
  spec and implementation plan.
