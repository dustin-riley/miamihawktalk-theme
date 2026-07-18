# MiamiHawkTalk Theme — Phase 1 Design ("Clean & Refined" Branded Theme)

**Date:** 2026-07-17
**Site:** https://miamihawktalk.fans (self-hosted Discourse on DigitalOcean, latest version)
**Community:** Miami University RedHawks athletics fan forum — motto "Love and Honor"

## Context

MiamiHawkTalk is a Discourse-based fan community currently running Discourse's
default look. The goal is a custom, unmistakably-RedHawks branded theme, built
and iterated collaboratively and delivered as a git-backed Discourse theme.

This is **Phase 1** of a phased plan:

- **Phase 1 (this spec):** Team-branded facelift — color schemes, logo/wordmark,
  fonts, header polish. Low risk, near-zero JS.
- **Phase 2 (future):** Distinctive fan-site feel — custom homepage/landing,
  restyled topic lists, banners (Glimmer `.gjs` components, `api.renderInOutlet`).
- **Phase 3 (future):** Fan features/widgets — schedule/scoreboard, game-day
  threads, featured content. LOE dominated by data sourcing, not styling.

## Goals (Phase 1)

- A cohesive RedHawks visual identity that reads as **clean & refined**: red used
  as a disciplined accent on a neutral canvas, not a wall of red.
- Original branding artwork carrying **no Miami University trademarked marks**.
- Light default + user-selectable dark, both legible and WCAG-conscious.
- Delivered as a full Discourse theme installable from a git repository — no
  server/droplet access required.

## Non-Goals (Phase 1)

- No custom homepage, banners, or landing area (Phase 2).
- No widgets, scoreboards, or live data (Phase 3).
- No Glimmer/`.gjs` components or `api.renderInOutlet` usage.
- No use of official Miami University logos, wordmarks, or trademarked imagery.

## Delivery Approach

- **Theme type:** Full theme (owns color schemes, header, fonts, logo). Chosen
  over a component because branded sites want top-level control of palette and
  identity, and every later phase layers on top of it.
- **Distribution:** Git repository. Installed via Admin → Customize → Themes →
  Install → *From a git repository*, then "Update" to pull new commits. No SSH
  into the droplet, no rebuilds, no database access.
- **Local iteration (optional):** Discourse Theme CLI (`gem install
  discourse_theme`) can watch the local folder and live-sync/preview against the
  site over an API key before changes touch the live default.

## Design Detail

### Color schemes

Red as a disciplined accent; neutral canvas carries the reading surface.

**Light (default)**
- Backgrounds: `#FFFFFF` primary, `#FAFAFA` secondary/canvas
- Text: `#1A1A1A` near-black
- Accent (RedHawks red): `#A6192E` — used on links, primary buttons, active nav,
  category markers, unread/notification badges only

**Dark (user-selectable in preferences)**
- Backgrounds: `#17191C` primary, `#202329` secondary
- Text: off-white (`#EDEDED`-range)
- Accent: `#E23A4E` — red brightened for WCAG-legible contrast on dark

Both are defined as named Discourse color schemes in `about.json` so users can
switch them from their profile. Exact red values are tunable once seen live.

### Typography

- **Body / UI:** Inter (bundled), with a system-sans fallback. Optimized for
  long-thread readability.
- **Wordmark / headings:** a tighter display weight of the same family for
  cohesion — refined rather than sporty-aggressive.

### Logo & branding (original artwork)

- **Desktop header:** SVG typographic wordmark — "MiamiHawkTalk" with a small
  "Love and Honor" tagline.
- **Mobile header + favicon:** compact original "MHT / hawk" monogram SVG.
- Wired via the theme's logo settings (large logo, small/mobile logo, favicon) so
  they render in the correct slots automatically.

### Header polish

Keep Discourse's familiar header layout (low risk). Refinements only:
- Cleaner spacing
- Red accent on the active nav item
- Thin red top hairline for identity

## Repo Structure

```
about.json            # manifest + both color schemes + settings declaration
settings.yml          # a few toggles (e.g. accent shade, show tagline)
common/common.scss    # shared styling
desktop/desktop.scss  # desktop-only tweaks
mobile/mobile.scss    # mobile-only tweaks
assets/               # wordmark.svg, monogram.svg, favicon
locales/en.yml        # human-readable setting labels
README.md             # install + local-dev instructions
LICENSE
```

## Verification

A theme is presentation, so verification means seeing it render:

1. **Live install:** push repo → install from git on the site → eyeball light and
   dark schemes on desktop and mobile.
2. **Preview-first (optional):** run through the Discourse Theme CLI preview
   before it becomes the site default, to iterate without affecting visitors.

Checks each iteration: light + dark legibility, header/logo rendering on desktop
and mobile, accent color contrast (WCAG AA for text/interactive elements), no
layout regressions in topic lists and topic view.

## Risks & Notes

- **Trademark:** All branding artwork is original. Official Miami University marks
  are intentionally excluded to avoid IP exposure for an unofficial fan site.
- **Upgrade safety:** Phase 1 is SCSS + color schemes + assets with essentially no
  JS, so it is resilient across Discourse upgrades. (Phase 2's Glimmer components
  will carry more upgrade sensitivity — tracked separately.)
- **Discourse version:** Target is the latest Discourse; color scheme + logo
  settings and SCSS overrides used here are stable across recent versions.
