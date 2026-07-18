# MiamiHawkTalk Theme Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a "clean & refined" RedHawks-branded full Discourse theme for miamihawktalk.fans, delivered as an installable git repo.

**Architecture:** A full Discourse theme = an `about.json` manifest declaring two color schemes (light default, dark optional) and image assets, plus SCSS files that use Discourse's scheme-derived color variables and inject the header wordmark/monogram. Near-zero JavaScript, so it is resilient across Discourse upgrades. Verification is structural (validate JSON/YAML/SVG/SCSS locally) plus a final visual pass after installing on the site.

**Tech Stack:** Discourse theme system (about.json, SCSS, SVG assets), git-based distribution. Local validation via `python3` (JSON), `xmllint` (SVG/XML), `ruby -ryaml` (YAML), and optional `npx sass` (SCSS syntax). No application framework.

## Global Constraints

- **Theme type:** Full theme (not a component) — owns color schemes, header, fonts, logo.
- **No Miami University IP:** all logo/brand artwork is original; no official RedHawk logo, beveled "M", or trademarked wordmark.
- **Accent discipline:** red is an accent only (links, primary buttons, active nav, category markers, unread/notification badges) on a neutral canvas — never a red-flooded surface.
- **Light accent red:** `#A6192E`. **Dark accent red:** `#E23A4E`. (Tunable later; use these verbatim as the starting values.)
- **Light backgrounds:** `#FFFFFF` primary, `#FAFAFA` secondary. **Light text:** `#1A1A1A`.
- **Dark backgrounds:** `#17191C` primary, `#202329` secondary. **Dark text:** `#EDEDED`.
- **Body/UI font:** Inter with a system-sans fallback. Headings/wordmark: heavier Inter weight.
- **Target Discourse:** latest. Use only stable color-scheme + SCSS + asset mechanisms; no Glimmer/`.gjs`, no `api.renderInOutlet` (those are Phase 2).
- **Favicon is a site setting**, not a theme file: produce the asset, upload it in Admin → Settings → Branding during install.
- **Commit after every task** with a clear message.

---

### Task 1: Repo scaffold + `about.json` with both color schemes

**Files:**
- Create: `about.json`
- Create: `.gitignore`

**Interfaces:**
- Produces: color scheme names `MiamiHawkTalk Light` and `MiamiHawkTalk Dark`; an `assets` block will be added to this same file in Task 3. Discourse maps scheme colors to SCSS variables `$primary` (text), `$secondary` (background), `$tertiary` (accent/links/buttons), `$header_background`, `$highlight`, `$danger`, `$success` — later SCSS tasks rely on these.

- [ ] **Step 1: Write `.gitignore`**

```gitignore
.discourse-site.json
node_modules/
*.log
.DS_Store
```

- [ ] **Step 2: Write `about.json` with manifest and both color schemes**

```json
{
  "name": "MiamiHawkTalk",
  "about_url": "https://miamihawktalk.fans",
  "license_url": "https://github.com/OWNER/miamihawktalk-theme/blob/main/LICENSE",
  "authors": "MiamiHawkTalk",
  "component": false,
  "color_schemes": {
    "MiamiHawkTalk Light": {
      "primary": "1A1A1A",
      "secondary": "FFFFFF",
      "tertiary": "A6192E",
      "quaternary": "A6192E",
      "header_background": "FFFFFF",
      "header_primary": "1A1A1A",
      "highlight": "FCE8EB",
      "danger": "A6192E",
      "success": "1E7A46",
      "love": "A6192E"
    },
    "MiamiHawkTalk Dark": {
      "primary": "EDEDED",
      "secondary": "17191C",
      "tertiary": "E23A4E",
      "quaternary": "E23A4E",
      "header_background": "202329",
      "header_primary": "EDEDED",
      "highlight": "3A2226",
      "danger": "E23A4E",
      "success": "3DB06E",
      "love": "E23A4E"
    }
  }
}
```

- [ ] **Step 3: Validate `about.json` is well-formed and has both schemes**

Run:
```bash
python3 -c "import json; d=json.load(open('about.json')); s=d['color_schemes']; assert 'MiamiHawkTalk Light' in s and 'MiamiHawkTalk Dark' in s; assert d['component'] is False; print('OK: schemes', list(s))"
```
Expected: `OK: schemes ['MiamiHawkTalk Light', 'MiamiHawkTalk Dark']`

- [ ] **Step 4: Commit**

```bash
git add about.json .gitignore
git commit -m "Add theme manifest with light and dark color schemes"
```

---

### Task 2: Original brand assets (wordmark, monogram, favicon)

**Files:**
- Create: `assets/wordmark.svg`
- Create: `assets/monogram.svg`
- Create: `assets/favicon.png` (rasterized from monogram; produced in Step 4)

**Interfaces:**
- Produces: asset files referenced by name in Task 3's `about.json` `assets` block (`wordmark`, `monogram`) and by the SCSS in Tasks 4–5 via the injected SCSS variables `$wordmark` and `$monogram`. `favicon.png` is uploaded manually in Branding (not referenced by theme code).

- [ ] **Step 1: Write `assets/wordmark.svg` — original typographic wordmark**

Original artwork: "MiamiHawkTalk" wordmark with a small "Love and Honor" tagline. Uses `currentColor` so SCSS can tint it per scheme. Keep it a clean, tight sans; no university marks.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 300 48" role="img" aria-label="MiamiHawkTalk — Love and Honor">
  <text x="0" y="30" font-family="Inter, -apple-system, Segoe UI, Roboto, sans-serif"
        font-size="28" font-weight="800" letter-spacing="-0.5" fill="currentColor">
    <tspan>Miami</tspan><tspan fill="#A6192E">Hawk</tspan><tspan>Talk</tspan>
  </text>
  <text x="1" y="44" font-family="Inter, -apple-system, Segoe UI, Roboto, sans-serif"
        font-size="10" font-weight="600" letter-spacing="2.5" fill="currentColor" opacity="0.75">
    LOVE AND HONOR
  </text>
</svg>
```

- [ ] **Step 2: Write `assets/monogram.svg` — original compact mark**

Original "MHT" monogram inside a rounded badge, for the mobile header and as the favicon source. No university imagery.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64" role="img" aria-label="MiamiHawkTalk">
  <rect x="2" y="2" width="60" height="60" rx="14" fill="#A6192E"/>
  <text x="32" y="41" text-anchor="middle"
        font-family="Inter, -apple-system, Segoe UI, Roboto, sans-serif"
        font-size="26" font-weight="800" letter-spacing="-1" fill="#FFFFFF">MHT</text>
</svg>
```

- [ ] **Step 3: Validate both SVGs are well-formed XML**

Run:
```bash
xmllint --noout assets/wordmark.svg assets/monogram.svg && echo "OK: SVGs valid"
```
Expected: `OK: SVGs valid` (no error output)

- [ ] **Step 4: Rasterize the monogram to a favicon PNG**

Try, in order, whichever renderer is installed. `rsvg-convert` (from `librsvg`) is preferred; fall back to ImageMagick, then macOS `qlmanage`/`sips`.

```bash
if command -v rsvg-convert >/dev/null; then
  rsvg-convert -w 512 -h 512 assets/monogram.svg -o assets/favicon.png
elif command -v magick >/dev/null; then
  magick -background none assets/monogram.svg -resize 512x512 assets/favicon.png
elif command -v convert >/dev/null; then
  convert -background none assets/monogram.svg -resize 512x512 assets/favicon.png
else
  echo "No SVG rasterizer found. Install librsvg (brew install librsvg) then rerun this step."; exit 1
fi
echo "created assets/favicon.png"
```
Expected: `created assets/favicon.png` and the file exists.

- [ ] **Step 5: Verify the PNG is a real image of the expected size**

Run:
```bash
python3 -c "import struct; f=open('assets/favicon.png','rb'); sig=f.read(8); assert sig==b'\x89PNG\r\n\x1a\n','not a PNG'; f.read(4); assert f.read(4)==b'IHDR'; w,h=struct.unpack('>II', f.read(8)); print('OK PNG', w,'x',h)"
```
Expected: `OK PNG 512 x 512`

- [ ] **Step 6: Commit**

```bash
git add assets/wordmark.svg assets/monogram.svg assets/favicon.png
git commit -m "Add original wordmark, monogram, and favicon assets"
```

---

### Task 3: Wire assets and settings into the theme

**Files:**
- Modify: `about.json` (add `assets` block)
- Create: `settings.yml`
- Create: `locales/en.yml`

**Interfaces:**
- Consumes: `assets/wordmark.svg`, `assets/monogram.svg` from Task 2; color schemes from Task 1.
- Produces: SCSS variables `$wordmark` and `$monogram` (auto-injected by Discourse from the `assets` block) used in Tasks 4–5. Setting keys `show_tagline` (bool) and `accent_color_light` / `accent_color_dark` (strings), referenced by SCSS in Task 4.

- [ ] **Step 1: Add an `assets` block to `about.json`**

Insert a top-level `"assets"` key (sibling of `"color_schemes"`). The full file becomes:

```json
{
  "name": "MiamiHawkTalk",
  "about_url": "https://miamihawktalk.fans",
  "license_url": "https://github.com/OWNER/miamihawktalk-theme/blob/main/LICENSE",
  "authors": "MiamiHawkTalk",
  "component": false,
  "assets": {
    "wordmark": "assets/wordmark.svg",
    "monogram": "assets/monogram.svg"
  },
  "color_schemes": {
    "MiamiHawkTalk Light": {
      "primary": "1A1A1A",
      "secondary": "FFFFFF",
      "tertiary": "A6192E",
      "quaternary": "A6192E",
      "header_background": "FFFFFF",
      "header_primary": "1A1A1A",
      "highlight": "FCE8EB",
      "danger": "A6192E",
      "success": "1E7A46",
      "love": "A6192E"
    },
    "MiamiHawkTalk Dark": {
      "primary": "EDEDED",
      "secondary": "17191C",
      "tertiary": "E23A4E",
      "quaternary": "E23A4E",
      "header_background": "202329",
      "header_primary": "EDEDED",
      "highlight": "3A2226",
      "danger": "E23A4E",
      "success": "3DB06E",
      "love": "E23A4E"
    }
  }
}
```

- [ ] **Step 2: Write `settings.yml`**

```yaml
show_tagline:
  type: bool
  default: true
accent_color_light:
  type: string
  default: "#A6192E"
accent_color_dark:
  type: string
  default: "#E23A4E"
```

- [ ] **Step 3: Write `locales/en.yml` with human-readable setting labels**

```yaml
en:
  theme_metadata:
    description: "Clean, refined RedHawks-branded theme for MiamiHawkTalk."
    settings:
      show_tagline: "Show the 'Love and Honor' tagline under the header wordmark on desktop."
      accent_color_light: "Accent color used in the light scheme (hex, e.g. #A6192E)."
      accent_color_dark: "Accent color used in the dark scheme (hex, e.g. #E23A4E)."
```

- [ ] **Step 4: Validate all three files**

Run:
```bash
python3 -c "import json; d=json.load(open('about.json')); assert d['assets']['wordmark'].endswith('wordmark.svg') and d['assets']['monogram'].endswith('monogram.svg'); print('OK about.json assets')"
ruby -ryaml -e "y=YAML.load_file('settings.yml'); raise 'missing show_tagline' unless y.key?('show_tagline'); puts 'OK settings.yml'"
ruby -ryaml -e "y=YAML.load_file('locales/en.yml'); raise 'bad locale' unless y.dig('en','theme_metadata','settings','show_tagline'); puts 'OK en.yml'"
```
Expected:
```
OK about.json assets
OK settings.yml
OK en.yml
```

- [ ] **Step 5: Commit**

```bash
git add about.json settings.yml locales/en.yml
git commit -m "Wire brand assets and theme settings into manifest"
```

---

### Task 4: Core styling — `common/common.scss`

**Files:**
- Create: `common/common.scss`

**Interfaces:**
- Consumes: SCSS color variables `$primary`, `$secondary`, `$tertiary`, `$header_background` (from Task 1 schemes); asset variable `$wordmark` (from Task 3); setting via `$show_tagline` is handled by Discourse but tagline visibility here is driven by the SVG itself + desktop rules in Task 5.
- Produces: shared styling relied upon (not overridden) by `desktop.scss` and `mobile.scss` in Task 5.

- [ ] **Step 1: Write `common/common.scss`**

Uses Inter with a system fallback, applies the accent to interactive elements, and replaces the default home logo with the wordmark SVG (tinted to the scheme's header text color via `currentColor` in the SVG). Keeps content surfaces neutral.

```scss
// ---- Typography ----
:root {
  --mht-font: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    Helvetica, Arial, sans-serif;
}

body,
.d-header,
input,
button,
select,
textarea {
  font-family: var(--mht-font);
}

h1, h2, h3, h4, h5, .title {
  font-family: var(--mht-font);
  font-weight: 700;
  letter-spacing: -0.01em;
}

// ---- Header wordmark ----
// Hide the default text/image logo and render our wordmark as a background.
.d-header #site-logo,
.d-header .title a img.logo-big,
.d-header .title h1 {
  // keep for a11y but visually replace with the wordmark
  font-size: 0;
  color: transparent;
}

.d-header .home-logo-wrapper-outlet,
.d-header .title a {
  display: inline-flex;
  align-items: center;
}

.d-header .title a::before {
  content: "";
  display: inline-block;
  width: 210px;
  height: 34px;
  background-image: url($wordmark);
  background-repeat: no-repeat;
  background-position: left center;
  background-size: contain;
  color: var(--header_primary); // feeds currentColor in the SVG text
}

// ---- Thin red identity hairline under the header ----
.d-header {
  border-top: 3px solid $tertiary;
}

// ---- Accent discipline: interactive elements only ----
a,
.d-header .icon:hover,
.topic-list .link-top-line a.title:hover {
  color: $tertiary;
}

.btn-primary {
  background: $tertiary;
  border-color: $tertiary;

  &:hover,
  &:focus {
    background: darken($tertiary, 6%);
    border-color: darken($tertiary, 6%);
  }
}

// Category color markers + unread/new badges lean on the accent
.badge-notification.unread-notifications,
.badge-notification.new-topic {
  background-color: $tertiary;
}

// Keep content canvas neutral (no red flooding)
.topic-body,
.cooked {
  color: $primary;
}
```

- [ ] **Step 2: Syntax-check the SCSS**

Discourse injects `$wordmark`, `$tertiary`, etc. at build time, so a standalone compile can't resolve them — this step checks **syntax only** (balanced braces, valid rules) by compiling against a tiny stub. Create the stub outside the theme so it is never shipped:

```bash
mkdir -p /tmp/mht-scss-check
cat > /tmp/mht-scss-check/_stub.scss <<'EOF'
$primary:#000; $secondary:#fff; $tertiary:#A6192E; $quaternary:#A6192E;
$header_background:#fff; $header_primary:#000; $highlight:#eee;
$danger:#A6192E; $success:#1E7A46; $love:#A6192E; $wordmark:"w.svg"; $monogram:"m.svg";
EOF
cat /tmp/mht-scss-check/_stub.scss common/common.scss > /tmp/mht-scss-check/check.scss
if command -v npx >/dev/null; then
  npx --yes sass --no-source-map /tmp/mht-scss-check/check.scss /tmp/mht-scss-check/out.css && echo "OK: common.scss compiles"
else
  echo "npx not available — skip compile; verify visually on install"
fi
```
Expected: `OK: common.scss compiles` (or the skip message if `npx` is unavailable — that is acceptable; visual verification in Task 6 covers it).

- [ ] **Step 3: Commit**

```bash
git add common/common.scss
git commit -m "Add core styling: Inter type, accent discipline, header wordmark"
```

---

### Task 5: Responsive rules — `desktop/desktop.scss` and `mobile/mobile.scss`

**Files:**
- Create: `desktop/desktop.scss`
- Create: `mobile/mobile.scss`

**Interfaces:**
- Consumes: `$tertiary` (Task 1), `$wordmark` and `$monogram` (Task 3), and the base header rules from `common/common.scss` (Task 4) — these files refine, not replace, those rules.
- Produces: final device-specific presentation. No downstream consumers.

- [ ] **Step 1: Write `desktop/desktop.scss`**

Full-size wordmark (with room for the tagline) and an accent underline on the active nav item.

```scss
// Larger wordmark on desktop; the SVG already includes the tagline line.
.d-header .title a::before {
  width: 250px;
  height: 42px;
}

// Active navigation item gets an accent underline
.nav-pills > li > a.active,
.list-controls .nav-pills > li > a.active {
  box-shadow: inset 0 -3px 0 $tertiary;
  color: $tertiary;
}
```

- [ ] **Step 2: Write `mobile/mobile.scss`**

Swap the wide wordmark for the compact monogram so the header stays tidy on small screens.

```scss
// Replace the wordmark with the square monogram on mobile.
.d-header .title a::before {
  width: 34px;
  height: 34px;
  background-image: url($monogram);
}
```

- [ ] **Step 3: Syntax-check both files against the stub**

```bash
for f in desktop/desktop.scss mobile/mobile.scss; do
  cat /tmp/mht-scss-check/_stub.scss "$f" > /tmp/mht-scss-check/check.scss
  if command -v npx >/dev/null; then
    npx --yes sass --no-source-map /tmp/mht-scss-check/check.scss /tmp/mht-scss-check/out.css \
      && echo "OK: $f compiles"
  else
    echo "npx not available — skip $f compile; verify visually"
  fi
done
```
Expected: `OK: desktop/desktop.scss compiles` and `OK: mobile/mobile.scss compiles` (or skip messages).

- [ ] **Step 4: Commit**

```bash
git add desktop/desktop.scss mobile/mobile.scss
git commit -m "Add responsive header rules: desktop wordmark + tagline, mobile monogram"
```

---

### Task 6: README, LICENSE, and full-theme structural validation

**Files:**
- Create: `README.md`
- Create: `LICENSE`

**Interfaces:**
- Consumes: every file from Tasks 1–5.
- Produces: the finished, installable theme repo. No downstream consumers.

- [ ] **Step 1: Write `LICENSE`**

Use the MIT license text with holder "MiamiHawkTalk" (year 2026). (Standard MIT body — copy the canonical text; artwork in `assets/` is original work covered by the same license.)

- [ ] **Step 2: Write `README.md`**

````markdown
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
````

- [ ] **Step 3: Validate the complete theme structure**

Confirms every required file exists and JSON/YAML are still valid after all edits.

```bash
python3 - <<'EOF'
import json, os, sys
required = ["about.json","settings.yml","locales/en.yml",
            "common/common.scss","desktop/desktop.scss","mobile/mobile.scss",
            "assets/wordmark.svg","assets/monogram.svg","assets/favicon.png",
            "README.md","LICENSE"]
missing = [f for f in required if not os.path.exists(f)]
assert not missing, f"MISSING: {missing}"
d = json.load(open("about.json"))
assert d["component"] is False
assert set(d["color_schemes"]) == {"MiamiHawkTalk Light","MiamiHawkTalk Dark"}
assert d["assets"]["wordmark"] and d["assets"]["monogram"]
print("OK: all", len(required), "files present; about.json valid")
EOF
```
Expected: `OK: all 11 files present; about.json valid`

- [ ] **Step 4: Commit**

```bash
git add README.md LICENSE
git commit -m "Add README and LICENSE; theme structure complete"
```

---

### Task 7: Install on the live site and visually verify

**Files:** none (verification only).

**Interfaces:**
- Consumes: the finished repo from Task 6.
- Produces: a confirmed-working installed theme. This is the true acceptance gate for a presentation-layer project.

- [ ] **Step 1: Push the repo to a remote**

```bash
git remote add origin <your-git-remote-url>   # e.g. git@github.com:OWNER/miamihawktalk-theme.git
git push -u origin main
```
Expected: branch `main` pushed to the remote.

- [ ] **Step 2: Install from git in Discourse**

In **Admin → Customize → Themes → Install → From a git repository**, paste the
repo URL and install. Do **not** set as default yet — keep it user-selectable
for review.

- [ ] **Step 3: Upload the favicon**

**Admin → Settings → Branding** → set **favicon** and **large icon** to the
`assets/favicon.png` produced in Task 2.

- [ ] **Step 4: Visual verification checklist**

Select the theme for your own account (Preferences → Interface) and confirm each
item, on both a desktop browser and a phone (or responsive mode):

- [ ] Header shows the **wordmark** on desktop; **monogram** on mobile.
- [ ] "Love and Honor" tagline is visible under the desktop wordmark.
- [ ] **Light scheme:** white/neutral canvas, `#A6192E` on links/buttons/active nav, thin red hairline under the header.
- [ ] **Dark scheme** (switch in Preferences → Interface): charcoal canvas, `#E23A4E` accent, text is comfortably legible (no low-contrast gray-on-charcoal).
- [ ] Active nav item shows the accent underline (desktop).
- [ ] Unread/new notification badges use the accent red.
- [ ] Topic list and an open topic look clean — content areas are **not** flooded with red.
- [ ] Favicon shows the monogram in the browser tab.

- [ ] **Step 5: Record results and set default**

Note anything off (colors to tune, sizing) for a follow-up pass. Once satisfied,
open the theme in Admin and **Set as default**. If issues remain, capture them as
the punch list for iteration — the git-update loop (push → Update) applies fixes.

---

## Self-Review

**Spec coverage:**
- Full theme (not component) → Global Constraints + Task 1 (`"component": false`). ✓
- Light default + dark selectable color schemes → Task 1. ✓
- Accent discipline (interactive only, neutral canvas) → Task 4. ✓
- Inter typography → Task 4. ✓
- Typographic wordmark (desktop) → Tasks 2, 4, 5. ✓
- Monogram (mobile + favicon) → Tasks 2, 5; favicon upload → Tasks 6–7. ✓
- Header polish (spacing, active-nav accent, red hairline) → Tasks 4–5. ✓
- No Miami University IP → Global Constraints + Task 2 (original SVGs). ✓
- Git-repo delivery + install instructions → Tasks 6–7. ✓
- Verification (light/dark, desktop/mobile) → Task 7 checklist. ✓
- Out of scope (homepage/widgets/Glimmer/live data) → excluded; not present in any task. ✓

**Correction vs spec:** the spec said favicon "wired into the theme"; favicon is
actually a Discourse **site setting**, so the plan produces the asset and uploads
it in Branding (Tasks 6–7). This is the accurate mechanism; called out explicitly.

**Placeholder scan:** no TBD/TODO/"handle edge cases"; MIT license and SVG bodies
are provided as concrete content. The only intentional user-supplied blanks are
the git remote URL and GitHub `OWNER` in `license_url` — unavoidable and flagged.

**Type/name consistency:** SCSS variable names (`$primary`, `$secondary`,
`$tertiary`, `$header_background`, `$wordmark`, `$monogram`) and color-scheme
names (`MiamiHawkTalk Light` / `MiamiHawkTalk Dark`) are used identically across
Tasks 1, 3, 4, 5, 6. Asset keys (`wordmark`, `monogram`) match between `about.json`
(Task 3) and the SCSS `url()` references (Tasks 4–5). ✓
