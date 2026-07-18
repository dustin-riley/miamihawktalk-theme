# MiamiHawkTalk Discourse Theme

A clean, refined RedHawks-branded **full theme** for
[miamihawktalk.fans](https://miamihawktalk.fans). Light default + optional dark,
original branding (no university trademarks), near-zero JavaScript.

## Install (from a git repository)

1. Push this repo to GitHub (public, or private with a deploy key).
2. In Discourse: **Admin → Customize → Themes → Install → From a git repository**.
3. Paste the repo URL and install.
4. Set it as the default: open the theme → **Set as default** (or leave it as a
   user-selectable theme first while you review).
5. Upload the favicon: **Admin → Settings → Branding** → set **favicon** and
   **large icon** to `assets/favicon.png` (the favicon is a Discourse site
   setting, not part of the theme files).

## Updating

Push new commits, then open the theme in Admin and click **Update**. Enable
"Auto-update" if you want it to pull automatically.

## Local development (optional)

```bash
gem install discourse_theme
discourse_theme watch .
```
This live-syncs local edits to a preview on your site via an API key.

## Color schemes

- **MiamiHawkTalk Light** (default) — neutral canvas, red `#A6192E` accent.
- **MiamiHawkTalk Dark** — charcoal canvas, red `#E23A4E` accent.

Users switch schemes in **Preferences → Interface**.

## Structure

- `about.json` — manifest, color schemes, asset declarations
- `settings.yml` / `locales/en.yml` — theme settings + labels
- `common|desktop|mobile/*.scss` — styling
- `assets/` — original wordmark, monogram, favicon
