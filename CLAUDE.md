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

## ⚠️ Editing styles? Load the `discourse-styling` skill first

Before touching any `.scss` file, use the **`discourse-styling`** skill. Discourse
has several non-obvious behaviors — most importantly that **SCSS color variables
do not adapt to the active color scheme** — that will otherwise silently produce
a broken dark mode. Every one of them has already caused a shipped bug here.

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
(2) looking at the running site. **Only the site can confirm CSS actually matches
Discourse's live DOM** — no offline tool catches a selector mismatch, so never
claim a visual fix works without seeing it rendered, in **both** color schemes.

```bash
python3 -m json.tool about.json > /dev/null       # manifest
ruby -ryaml -e "YAML.load_file('locales/en.yml')" # locales
xmllint --noout assets/*.svg                      # artwork
```

Regenerate the favicon after changing the monogram:

```bash
rsvg-convert -w 512 -h 512 assets/monogram.svg -o assets/favicon.png
```

(SCSS syntax-checking needs a variable stub — see the `discourse-styling` skill.)

## Architecture

- **`about.json`** — the manifest. Defines the two color schemes
  (`MiamiHawkTalk Light`, `MiamiHawkTalk Dark`) and declares assets. Assets
  become SCSS variables of the same name (`wordmark` → `$wordmark`).
- **`common/` `desktop/` `mobile/`** — SCSS. `common/` applies everywhere; the
  other two refine it. Mobile swaps the wordmark for the monogram.
- **`assets/`** — original SVG artwork plus the generated favicon.
- **`locales/en.yml`** — theme metadata strings.

**The favicon is a Discourse site setting, not a theme file.** `assets/favicon.png`
is produced here but must be uploaded manually under **Admin → Settings → Branding**.
The theme cannot set it.

## Branding constraints

All artwork in `assets/` is **original**. Miami University's actual trademarks
(the RedHawk logo, beveled "M", official wordmarks) are deliberately **not** used
— this is an unofficial fan site. Do not introduce university marks.

## Project docs

- `docs/superpowers/` — this repo's design spec and implementation plan.
  Athletics schedule documents live in the plugin repo, not here.
- **Planned work lives in `../BACKLOG.md`**, one level up, shared across all
  three MiamiHawkTalk repos. Todos do not belong in this file.

**Phase 1 is deliberately near-zero JavaScript** — no `.gjs`, no
`api.renderInOutlet` — which is why it is robust across Discourse upgrades.
Adding JS to this repo moves work into Phase 2 scope; the schedule sidebar
deliberately lives in a separate component repo for exactly that reason.
