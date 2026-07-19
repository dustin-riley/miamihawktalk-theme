# MiamiHawkTalk Workspace Restructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Contain the three MiamiHawkTalk repositories under one parent folder, give each a `CLAUDE.md` holding only what every agent in that repo needs, and merge the scattered backlogs into one version-controlled list.

**Architecture:** A minimal git repo at `~/Development/MiamiHawkTalk/` tracks planning artifacts only — `CLAUDE.md`, `BACKLOG.md`, `docs/superpowers/` — and gitignores the three child repos, which stay fully independent with their own remotes and history. Moves are plain `mv`; no git history is rewritten anywhere.

**Tech Stack:** Markdown, git, shell. No code changes.

**Spec:** `docs/superpowers/specs/2026-07-19-workspace-restructure-design.md` (moves to the parent repo in Task 7)

## Global Constraints

- Parent folder: `/Users/dustin/Development/MiamiHawkTalk/`
- The three child repos keep their existing directory names: `miamihawktalk-theme`, `discourse-redhawks-schedule`, `miamihawktalk-schedule`
- **`CLAUDE.md` holds only what EVERY agent touching that folder needs.** Anything conditional becomes a skill instead.
- **No todos in any `CLAUDE.md`.** Backlog items live in `BACKLOG.md` only.
- The parent `CLAUDE.md` must NOT contain per-repo deploy mechanics or runtime architecture — those live in each child.
- **`miamihawktalk-theme` moves LAST.** It is the running session's working directory; moving it earlier breaks subsequent commands.
- **`docs/ROADMAP.md` is deleted in the same commit that its merged content lands** in `BACKLOG.md`, so content is never in limbo.
- No child repo's git history, remote, or deployed code is touched. Discourse clones from GitHub, so the live site is unaffected throughout.
- Do not create empty `TODO.md` files in child repos — `BACKLOG.md` is the single list.

---

### Task 1: Create the parent repo

**Files:**
- Create: `/Users/dustin/Development/MiamiHawkTalk/.gitignore`

**Interfaces:**
- Produces: the parent repo that every later task commits into.

- [ ] **Step 1: Create and initialise**

```bash
mkdir -p /Users/dustin/Development/MiamiHawkTalk
cd /Users/dustin/Development/MiamiHawkTalk
git init
```

- [ ] **Step 2: Write `.gitignore`**

The three child repos are independent and must never be tracked here.

```
# The child repos are independent git repositories with their own remotes.
# This parent tracks planning artifacts only.
miamihawktalk-theme/
discourse-redhawks-schedule/
miamihawktalk-schedule/

.DS_Store
```

- [ ] **Step 3: Commit**

```bash
cd /Users/dustin/Development/MiamiHawkTalk
git add .gitignore
git commit -m "chore: initialise MiamiHawkTalk workspace"
```

Expected: one commit, one file.

---

### Task 2: Move the two schedule repos in

The theme is deliberately excluded — it moves in Task 8.

**Files:** no file contents change; two directories move.

- [ ] **Step 1: Move both repos**

```bash
mv /Users/dustin/Development/discourse-redhawks-schedule /Users/dustin/Development/MiamiHawkTalk/
mv /Users/dustin/Development/miamihawktalk-schedule /Users/dustin/Development/MiamiHawkTalk/
```

- [ ] **Step 2: Verify both repos survived intact**

```bash
for r in discourse-redhawks-schedule miamihawktalk-schedule; do
  echo "=== $r ==="
  git -C /Users/dustin/Development/MiamiHawkTalk/$r log --oneline -1
  git -C /Users/dustin/Development/MiamiHawkTalk/$r remote -v | head -1
  git -C /Users/dustin/Development/MiamiHawkTalk/$r status --porcelain
done
```

Expected: each prints its most recent commit, an `origin` remote pointing at `github.com:dustin-riley/<repo>.git`, and **no** uncommitted changes. A moved git repo is self-contained, so nothing should differ.

- [ ] **Step 3: Verify the parent does not try to track them**

```bash
cd /Users/dustin/Development/MiamiHawkTalk && git status --porcelain
```

Expected: **empty output.** If either repo directory appears, `.gitignore` is wrong — fix it before continuing, or the parent will try to swallow both child repos.

- [ ] **Step 4: Confirm the plugin's tests still run from the new location**

```bash
cd /Users/dustin/Development/MiamiHawkTalk/discourse-redhawks-schedule
~/.gem/ruby/2.6.0/bin/rspec spec/lib/parser_spec.rb
```

Expected: `32 examples, 0 failures`. The specs use `__dir__`-relative fixture paths, so the move must not affect them — this proves it.

No commit: nothing tracked changed.

---

### Task 3: Plugin `CLAUDE.md`

**Files:**
- Create: `/Users/dustin/Development/MiamiHawkTalk/discourse-redhawks-schedule/CLAUDE.md`

- [ ] **Step 1: Write the file**

Every item below is something *every* agent working in this repo needs. Deploy cost, the peer-auth trap and the Ruby version mismatch have each already cost real time.

````markdown
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **Discourse plugin** that fetches the Miami University Athletics RSS calendar
server-side every 30 minutes and serves upcoming events at
`/redhawks-schedule.json`.

It exists because `miamiredhawks.com` sends no CORS headers, so a browser cannot
fetch that feed directly. The companion theme component
(`../miamihawktalk-schedule/`) renders this JSON in the sidebar.

## Deploying costs downtime

Unlike the theme-side repos, changes here require a container rebuild:

```bash
cd /var/discourse && ./launcher rebuild app
```

That is several minutes with the site down, so batch changes rather than
deploying one at a time. If the site fails to come back, remove this plugin's
clone line from `containers/app.yml` and rebuild again — that restores service
immediately. Debug afterwards, not during.

## Running anything against the live app

`rails runner` **must run as the `discourse` user.** Postgres uses peer
authentication and rejects root with "Peer authentication failed for user
discourse".

```bash
cd /var/discourse && ./launcher enter app
cd /var/www/discourse
sudo -E -u discourse bundle exec rails runner 'puts PluginStore.get("discourse-redhawks-schedule", "events")["events"].length'
```

## Tests

```bash
~/.gem/ruby/2.6.0/bin/rspec spec/lib/parser_spec.rb
```

`rspec` is installed user-scoped and is **not** on `PATH`.

**Local Ruby is 2.6.10; the container runs 3.x.** Syntax newer than 2.6 —
`filter_map`, `Hash#except`, endless method definitions, rightward assignment —
works in production and fails only on this machine. Stay 2.6-compatible.

## Architecture

- **`lib/redhawks_schedule/parser.rb`** — pure Ruby, with **no Rails, HTTP or
  persistence**. That is deliberate: it puts all the interesting logic somewhere
  unit testable without a Discourse checkout. Keep it that way.
- **`app/jobs/scheduled/fetch_redhawks_schedule.rb`** — Sidekiq, every 30
  minutes. Failures are logged and swallowed, and a response not containing
  `<rss` is discarded rather than stored — so an upstream outage leaves the last
  good data in place and stays invisible to users.
- **`app/controllers/redhawks_schedule_controller.rb`** — serves the stored
  payload. `requires_login false` is load-bearing: the sidebar renders for
  anonymous visitors too.
- Storage is **`PluginStore`**, not a table. The payload is a few KB and needs no
  querying, so a migration would be overhead with no benefit.

## The feed's dominant quirk

**59% of events carry no announced time.** The feed expresses this as a
date-only `<ev:startdate>` (`2026-08-29`) rather than the string "TBA", and
those are **Eastern calendar dates** that must never be timezone-converted.
Hockey and football are the most affected sports. `time_known` in the payload is
what marks them.

## Docs

- `NOTES-api-verification.md` — Discourse API facts verified against source
  (outlet names, helper signatures), with where each came from.
- `docs/superpowers/` — the schedule design spec and implementation plan.
- Planned work lives in `../BACKLOG.md`, not here.
````

- [ ] **Step 2: Commit**

```bash
cd /Users/dustin/Development/MiamiHawkTalk/discourse-redhawks-schedule
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md"
```

---

### Task 4: Component `CLAUDE.md`

**Files:**
- Create: `/Users/dustin/Development/MiamiHawkTalk/miamihawktalk-schedule/CLAUDE.md`

- [ ] **Step 1: Write the file**

The `themePrefix` rule and the client/server grace-window contract are both here because each was a real shipped defect caught in review.

````markdown
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **Discourse theme component** that renders the "Upcoming Games" block in the
sidebar of miamihawktalk.fans. It reads `/redhawks-schedule.json`, which the
companion plugin (`../discourse-redhawks-schedule/`) serves.

There is no build step, no package manager and no test runner. The deliverable
is the source files themselves.

## Deploy / iteration loop

```bash
git push origin main
```

Then in Discourse: **Admin → Customize → Themes → Components → MiamiHawkTalk
Schedule → Update**, and hard-refresh (Cmd+Shift+R). No rebuild and no downtime
— which is exactly why this is a separate repo from the plugin.

## Translations must go through `themePrefix`

Theme components do **not** use the plugin locale convention. Keys sit directly
under `en:` with no `js:` level, and Discourse stores them as
`theme_translation.<theme_id>.<key>`. They resolve only via `themePrefix`:

```js
import { i18n } from "discourse-i18n";
import { themePrefix } from "virtual:theme";

i18n(themePrefix("title"));        // in JS
```

```hbs
{{i18n (themePrefix "title")}}     {{!-- in a template --}}
```

A bare `i18n("title")` renders the literal text `[en.title]` on screen for every
user. This shipped once and was caught only in final review.

## Rendering

The component renders into the **`after-sidebar-sections`** plugin outlet via
`api.renderInOutlet`. `api.addSidebarSection` is not usable here: it is
link-based, one line of text per link, and cannot express the two-line row.

**When there is no data, render nothing** — no error text, no empty container,
no heading. Zero upcoming events is the normal state every June, and a
persistent error box in the sidebar for six weeks is worse than absence.

## Colours come from the parent theme

`common/common.scss` uses CSS custom properties: `var(--primary)`,
`var(--tertiary)`, `var(--primary-medium)`, `var(--primary-low)`. **Never
hardcode a colour.** The parent theme ships both a light and a dark scheme, and
a literal hex breaks one of them.

## The client/server filtering contract

`upcomingOnly()` in `javascripts/discourse/lib/schedule-data.js` re-filters past
events on the client. Its `DATE_ONLY_GRACE_MS` **must match** `DATE_ONLY_GRACE`
in the plugin's parser — currently 30 hours. When they disagreed, untimed games
vanished from the sidebar at 8pm Eastern on the very day they were played.

## Verifying changes

There are no unit tests. Verification is looking at the running site, in **both**
colour schemes, logged in **and** logged out. Only the site can confirm CSS
matches Discourse's live DOM.

Structural checks that do work offline:

```bash
python3 -m json.tool about.json > /dev/null       # manifest
ruby -ryaml -e "YAML.load_file('locales/en.yml')" # locales
node --check javascripts/discourse/lib/schedule-data.js
```

`.gjs` files cannot be checked with `node --check` — their `<template>` syntax
is not JavaScript. Verify those by inspection: every component, helper and
modifier used in a template needs an explicit import, because `.gjs` templates
are strict-mode.

Planned work lives in `../BACKLOG.md`, not here.
````

- [ ] **Step 2: Commit**

```bash
cd /Users/dustin/Development/MiamiHawkTalk/miamihawktalk-schedule
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md"
```

---

### Task 5: Relocate the athletics docs to the plugin repo

The schedule spec and plan currently sit in the theme repo but describe the plugin and component. The plan also hardcodes 39 absolute paths that the move invalidates.

**Files:**
- Create: `.../discourse-redhawks-schedule/docs/superpowers/specs/2026-07-18-athletics-schedule-design.md`
- Create: `.../discourse-redhawks-schedule/docs/superpowers/plans/2026-07-18-athletics-schedule.md`
- Delete: the same two files from `/Users/dustin/Development/miamihawktalk-theme/docs/superpowers/`

- [ ] **Step 1: Move the two files**

```bash
cd /Users/dustin/Development/MiamiHawkTalk/discourse-redhawks-schedule
mkdir -p docs/superpowers/specs docs/superpowers/plans
git -C /Users/dustin/Development/miamihawktalk-theme mv \
  docs/superpowers/specs/2026-07-18-athletics-schedule-design.md \
  /Users/dustin/Development/MiamiHawkTalk/discourse-redhawks-schedule/docs/superpowers/specs/ 2>/dev/null \
  || mv /Users/dustin/Development/miamihawktalk-theme/docs/superpowers/specs/2026-07-18-athletics-schedule-design.md \
        docs/superpowers/specs/
mv /Users/dustin/Development/miamihawktalk-theme/docs/superpowers/plans/2026-07-18-athletics-schedule.md \
   docs/superpowers/plans/
```

`git mv` across repository boundaries does not work, so a plain `mv` is used and each repo commits its own side.

- [ ] **Step 2: Fix the stale absolute paths**

The plan references the pre-move locations 39 times (24 plugin, 15 component). Rewrite them to the new parent-folder paths:

```bash
cd /Users/dustin/Development/MiamiHawkTalk/discourse-redhawks-schedule
python3 - <<'PY'
p = "docs/superpowers/plans/2026-07-18-athletics-schedule.md"
s = open(p).read()
before = s.count("/Users/dustin/Development/")
s = s.replace("/Users/dustin/Development/discourse-redhawks-schedule",
              "/Users/dustin/Development/MiamiHawkTalk/discourse-redhawks-schedule")
s = s.replace("/Users/dustin/Development/miamihawktalk-schedule",
              "/Users/dustin/Development/MiamiHawkTalk/miamihawktalk-schedule")
s = s.replace("/Users/dustin/Development/miamihawktalk-theme",
              "/Users/dustin/Development/MiamiHawkTalk/miamihawktalk-theme")
open(p, "w").write(s)
print("rewrote", before, "path references")
PY
```

Expected: `rewrote 39 path references`.

- [ ] **Step 3: Verify no stale paths remain**

```bash
grep -n "Development/discourse-redhawks-schedule\|Development/miamihawktalk-" \
  docs/superpowers/plans/2026-07-18-athletics-schedule.md | grep -v "MiamiHawkTalk/" || echo "clean"
```

Expected: `clean`. Any hit is a path that skips the `MiamiHawkTalk/` level.

- [ ] **Step 4: Commit the plugin side**

```bash
cd /Users/dustin/Development/MiamiHawkTalk/discourse-redhawks-schedule
git add docs
git commit -m "docs: adopt the athletics schedule spec and plan

Moved from the theme repo, which held documents describing this repo and
the theme component. Absolute paths updated for the new workspace layout."
```

- [ ] **Step 5: Commit the removal on the theme side**

```bash
cd /Users/dustin/Development/miamihawktalk-theme
git add -A docs
git commit -m "docs: hand the athletics schedule spec and plan to the plugin repo

They describe the plugin and theme component, not this repo."
```

---

### Task 6: Merged `BACKLOG.md` at the parent

**Files:**
- Create: `/Users/dustin/Development/MiamiHawkTalk/BACKLOG.md`
- Delete: `/Users/dustin/Development/miamihawktalk-theme/docs/ROADMAP.md` (Task 7)

**Interfaces:**
- Consumes: the five tracks, sequencing and open questions from `docs/ROADMAP.md`.
- Produces: the single backlog every `CLAUDE.md` points at.

- [ ] **Step 1: Write the file**

Carry the tracks over intact. Three substantive updates: Track D's schedule widget is now substantially delivered, the schedule build's deferred items are added, and the feed open-question is answered.

````markdown
# MiamiHawkTalk — Backlog

The single list of planned work across all three repos. Grouped by track, tagged
with rough effort, the repo it lands in, and whether it needs an external feed.
Nothing here is committed scope — it is the menu we pick from.

**Repos:** `[theme]` miamihawktalk-theme · `[plugin]` discourse-redhawks-schedule
· `[component]` miamihawktalk-schedule · `[new]` needs a new repo
**Kind:** `[style]` styling only · `[glimmer]` custom JS components ·
`[feature]` meaningful build · `[automation]` scheduled/AI job · `[data]` needs
an external feed
**Effort:** 🟢 small · 🟡 medium · 🔴 large

---

## Open now — finish the schedule sidebar

Live verification still outstanding from the 2026-07-19 deploy:

- [ ] Check a **TBA row against miamiredhawks.com** — confirm the date matches
      exactly. This is the one bug class the no-timezone-conversion rule exists
      to prevent, and it is invisible without comparing to the source.
      `[component]` 🟢
- [ ] Verify in the **dark colour scheme** `[component]` 🟢
- [ ] Verify **logged out** — anonymous is a separate server code path
      `[component]` 🟢
- [ ] Verify on **mobile** — no wrap or overflow at narrow widths
      `[component]` 🟢

Deferred by decision during the build:

- [ ] **Stale-tab re-filter** — the component fetches once in its constructor and
      the outlet persists across SPA navigation, so a tab left open all day keeps
      showing games that already started. Fresh loads are bounded at ~75 min
      staleness; open tabs are unbounded. `[component]` 🟢
- [ ] **Strip the `(Exhibition)` suffix** from opponent names — the feed puts it
      inside `s:opponent`, so rows truncate mid-word. The plugin already parses
      `s:gamepromoname` separately and the component ignores it. `[component]` 🟢

## ✅ Phase 1 — Branded theme (DONE)

- [x] Full theme: light/dark RedHawks colour schemes
- [x] Original wordmark + monogram + favicon
- [x] Inter typography, accent discipline, header polish
- [x] Installed at github.com/dustin-riley/miamihawktalk-theme
- [ ] Final sign-off: confirm dark-mode legibility + set as site default
      `[theme]` 🟢

## 🏟️ Track A — Game-day energy

- [ ] **Game-day mode** — auto banner on game days (matchup, kickoff, TV,
      live-thread link); hidden off-days. `[theme]`+`[automation]` 🟡
      (hand-set version 🟢; auto-detecting can now use the schedule plugin)
- [ ] **Auto-created game threads** — scheduled agent opens a pinned
      "Game Thread: Miami vs X" before each game. `[automation]` 🟡
- [ ] **Live score bug** — small scoreboard in header/sidebar during games.
      `[glimmer]`+`[data]` 🔴 (the RSS calendar carries no scores — needs a
      second source)
- [ ] **Post-game "Grade the RedHawks"** — auto-posted poll after each game,
      results roll up over the season. `[automation]`+`[feature]` 🟡

## 🎯 Track B — Fandom & competition

- [ ] **Season-long Pick'em + leaderboard** — weekly predictions, points,
      standings. Stickiest daily-return driver. `[feature]`+`[data]` 🔴
- [ ] **MAC Tournament / March bracket challenge** — basketball bracket pool.
      `[feature]` 🟡
- [ ] **Prediction-cred badges** — flair for accurate callers; builds on
      Pick'em. `[feature]` 🟡

## 🦅 Track C — Miami heritage (unique to this school)

- [ ] **"Cradle of Coaches" touches** — rotating quote / heritage strip / about
      page. `[theme]` 🟢
- [ ] **"This Day in RedHawks History"** — automated daily post or sidebar
      snippet. `[automation]` 🟡 (needs a content source)
- [ ] **Rivalry-week takeover** — Victory Bell (vs Cincinnati) / MAC rivalry
      banner + countdown. `[theme]`+`[automation]` 🟡
- [ ] **"Love and Honor" welcome ritual** — branded new-member greeting.
      `[theme]` 🟢

## 🏠 Track D — A real front door (Phase 2 core)

- [ ] **Custom homepage** — featured hero, schedule/record strip, trending,
      recruiting headlines. `[glimmer]` 🔴
- [~] **Schedule & record widget** — **substantially delivered** 2026-07-19 as
      the "Upcoming Games" sidebar block (`[plugin]` + `[component]`). Still
      open: **W/L record** and a **next-game countdown**, neither of which the
      RSS calendar provides — both need a results source. 🟡
- [ ] **Recruiting / commit tracker** — structured board for commits & targets.
      `[feature]` 🔴

## 🤖 Track E — AI-powered

- [ ] **"Catch me up"** — on-demand summary of a long game thread.
      `[automation]` 🟡
- [ ] **Weekly "Week Ahead" digest** — auto-posted every Monday. `[automation]`
      🟢 (can read the schedule plugin's JSON directly)
- [ ] **Auto game recaps** — generated thread after each game.
      `[automation]`+`[data]` 🟡

---

## Suggested sequencing

1. **Close out the schedule sidebar** — the verification items at the top are
   minutes of work and the feature is otherwise done.
2. **Close Phase 1** (dark-mode check + set default).
3. **Phase 2 = Track D custom homepage** — the biggest visual leap; makes the
   site feel like a destination.
4. **First automation win = Track E "Week Ahead" digest** — 🟢 effort, high
   visibility, and it can now read the schedule plugin's JSON rather than
   needing a new feed. Proves out the scheduled-agent pattern.
5. **Signature feature = Track A game-day mode** — the hand-set version first;
   auto-detection can use the schedule plugin.
6. **Big engagement bet = Track B Pick'em** — most work, most payoff; do once
   the foundations above are in.

## Open questions

- ~~Is there a reliable, allowed feed for Miami (OH) schedule + scores?~~
  **Answered 2026-07-19.** The athletics department publishes a SIDEARM RSS
  calendar at `https://miamiredhawks.com/calendar.ashx/calendar.rss`, now in
  production via the plugin. Caveat: it carries **schedule only, no scores or
  results** — every item above needing W/L still needs a second source.
- Which sports are in scope for game-day/Pick'em (football + men's basketball
  first, or all)? The schedule sidebar currently shows **all** sports.
- Any existing plugins installed (polls, etc.) we can build on?
````

- [ ] **Step 2: Commit**

```bash
cd /Users/dustin/Development/MiamiHawkTalk
git add BACKLOG.md
git commit -m "docs: merge roadmap and open items into one backlog"
```

---

### Task 7: Trim the theme repo and retire `ROADMAP.md`

**Files:**
- Modify: `/Users/dustin/Development/miamihawktalk-theme/CLAUDE.md:85-93`
- Delete: `/Users/dustin/Development/miamihawktalk-theme/docs/ROADMAP.md`
- Move: the workspace spec and plan into the parent repo

- [ ] **Step 1: Replace the "Project docs" section**

The current section describes the roadmap's phases in detail. Under the no-todos rule it becomes a pointer. The near-zero-JavaScript note is **guidance, not a todo** — it explains why this repo has no `.gjs` — so it stays.

Replace lines 85-93 of `CLAUDE.md` with:

```markdown
## Project docs

- `docs/superpowers/` — this repo's design spec and implementation plan.
  Athletics schedule documents live in the plugin repo, not here.
- **Planned work lives in `../BACKLOG.md`**, one level up, shared across all
  three MiamiHawkTalk repos. Todos do not belong in this file.

**Phase 1 is deliberately near-zero JavaScript** — no `.gjs`, no
`api.renderInOutlet` — which is why it is robust across Discourse upgrades.
Adding JS to this repo moves work into Phase 2 scope; the schedule sidebar
deliberately lives in a separate component repo for exactly that reason.
```

- [ ] **Step 2: Delete `ROADMAP.md` and move the workspace docs**

```bash
cd /Users/dustin/Development/miamihawktalk-theme
rm docs/ROADMAP.md

mkdir -p /Users/dustin/Development/MiamiHawkTalk/docs/superpowers/specs \
         /Users/dustin/Development/MiamiHawkTalk/docs/superpowers/plans
mv docs/superpowers/specs/2026-07-19-workspace-restructure-design.md \
   /Users/dustin/Development/MiamiHawkTalk/docs/superpowers/specs/
mv docs/superpowers/plans/2026-07-19-workspace-restructure.md \
   /Users/dustin/Development/MiamiHawkTalk/docs/superpowers/plans/
```

- [ ] **Step 3: Verify nothing still points at `ROADMAP.md`**

```bash
grep -rn "ROADMAP" /Users/dustin/Development/miamihawktalk-theme --include=*.md || echo "clean"
```

Expected: `clean`.

- [ ] **Step 4: Commit both sides**

The deletion and its replacement land together, so the content is never in limbo — `BACKLOG.md` was committed in Task 6.

```bash
cd /Users/dustin/Development/miamihawktalk-theme
git add -A
git commit -m "docs: point at the shared backlog and drop ROADMAP.md

Content merged into ../BACKLOG.md in the workspace repo. Todos do not
belong in CLAUDE.md, and the roadmap spanned all three repos rather than
this one."

cd /Users/dustin/Development/MiamiHawkTalk
git add docs
git commit -m "docs: adopt the workspace restructure spec and plan"
```

---

### Task 8: Parent `CLAUDE.md`, then move the theme last

**Files:**
- Create: `/Users/dustin/Development/MiamiHawkTalk/CLAUDE.md`

- [ ] **Step 1: Write the parent `CLAUDE.md`**

Routing and shared facts only. No deploy mechanics — those differ sharply between the plugin and the two theme-side repos and belong with each.

````markdown
# CLAUDE.md

This folder holds everything for **miamihawktalk.fans**, an unofficial Miami
University RedHawks fan forum running self-hosted Discourse.

**Nothing is built at this level.** This folder contains planning documents and
three independent git repositories. Make changes *inside* the relevant repo, and
follow that repo's own `CLAUDE.md` — it governs. This file never duplicates one.

## Which repo owns what

| Repo | Owns | Go here when |
|---|---|---|
| `miamihawktalk-theme/` | The full Discourse theme: colour schemes, header, wordmark, site-wide SCSS | Changing how the site looks or is branded |
| `discourse-redhawks-schedule/` | Discourse **plugin**: fetches the athletics RSS calendar server-side, serves `/redhawks-schedule.json` | Changing what schedule data exists, or how it is parsed |
| `miamihawktalk-schedule/` | Discourse **theme component**: renders the "Upcoming Games" sidebar block from that JSON | Changing how the schedule looks or behaves in the sidebar |

The plugin and the component are split on purpose: deploying the plugin needs a
container rebuild with real downtime, while the component updates with a git
pull. Anything likely to be iterated on belongs on the component side.

## Shared facts

- Site: `miamihawktalk.fans` — self-hosted Discourse, currently **2026.7.0-latest**
- All three repos deploy from GitHub under `dustin-riley/`, never from this disk
- This is an **unofficial** fan site. Miami University's trademarks — the RedHawk
  logo, the beveled "M", official wordmarks — are never used. All artwork is
  original.

## Planned work

`BACKLOG.md` is the single list across all three repos. Todos belong there and
never in a `CLAUDE.md`.

## Workspace docs

`docs/superpowers/` holds designs that span repos. Per-repo specs live in the
repo they describe.
````

- [ ] **Step 2: Commit it**

```bash
cd /Users/dustin/Development/MiamiHawkTalk
git add CLAUDE.md
git commit -m "docs: add workspace routing CLAUDE.md"
```

- [ ] **Step 3: Move the theme repo in — this must be last**

This invalidates the running session's working directory. Do nothing else in this session afterwards except the verification in Step 4.

```bash
mv /Users/dustin/Development/miamihawktalk-theme /Users/dustin/Development/MiamiHawkTalk/
```

- [ ] **Step 4: Verify the finished workspace**

```bash
cd /Users/dustin/Development/MiamiHawkTalk

echo "=== parent tracks planning artifacts only ==="
git status --porcelain            # expect: empty
git ls-files                      # expect: .gitignore, BACKLOG.md, CLAUDE.md, docs/...

echo "=== all three child repos intact ==="
for r in miamihawktalk-theme discourse-redhawks-schedule miamihawktalk-schedule; do
  printf "%-32s %s  %s\n" "$r" \
    "$(git -C $r log --oneline -1 | cut -c1-7)" \
    "$(git -C $r remote get-url origin)"
  git -C $r status --porcelain
done

echo "=== every repo has a CLAUDE.md ==="
ls CLAUDE.md */CLAUDE.md

echo "=== plugin tests still pass from the final location ==="
~/.gem/ruby/2.6.0/bin/rspec discourse-redhawks-schedule/spec/lib/parser_spec.rb 2>&1 | tail -2

echo "=== no path skips the MiamiHawkTalk level ==="
grep -rn "Development/miamihawktalk-\|Development/discourse-redhawks" \
  --include=*.md . | grep -v "MiamiHawkTalk/" || echo "clean"
```

Expected: parent `git status` empty; four `CLAUDE.md` files; each child on its own remote with a clean tree; `32 examples, 0 failures`; and `clean` from the final grep.

- [ ] **Step 5: Restart Claude in the new location**

```bash
cd /Users/dustin/Development/MiamiHawkTalk
```

The real test of this restructure is behavioural: start an agent here and ask for a change to the sidebar row layout. It should go to `miamihawktalk-schedule/` without further prompting. If it does not, the parent `CLAUDE.md` routing table needs sharpening.

---

## Self-Review

**Spec coverage:**

| Spec requirement | Task |
|---|---|
| Parent folder as a minimal git repo | 1 |
| Child repos gitignored, stay independent | 1, 2 (Step 3) |
| Move the two schedule repos | 2 |
| Plugin `CLAUDE.md` | 3 |
| Component `CLAUDE.md` | 4 |
| Schedule spec + plan to the plugin repo | 5 |
| Fix stale absolute paths | 5 (Steps 2-3) |
| Merged `BACKLOG.md` at the parent | 6 |
| Track D marked substantially delivered | 6 |
| Deferred + unfinished-verification items captured | 6 |
| Theme `CLAUDE.md` trimmed, roadmap pointer | 7 |
| `ROADMAP.md` deleted only once merged | 6 then 7 |
| Workspace docs to the parent repo | 7 (Step 2) |
| Parent `CLAUDE.md`: routing + shared facts, no deploy mechanics | 8 |
| Theme moves last | 8 (Step 3) |
| Verification of remotes, history, tracking | 2, 8 (Step 4) |
| No empty `TODO.md` files | Not created anywhere |

**Placeholder scan:** none. Every file's full contents are given inline.

**Consistency:** every `CLAUDE.md` points at `../BACKLOG.md` (children) or
`BACKLOG.md` (parent), matching the file created in Task 6. The parent
`CLAUDE.md` references `docs/superpowers/`, created in Task 7 Step 2 — which
runs before Task 8 Step 1 writes that reference.

**Ordering risk:** Task 5 and Task 7 both commit in the theme repo, and Task 7
moves this plan file out of it. The plan is read from the executing agent's
context, not from disk, so relocating it mid-execution is safe — but Task 7 Step
2 must run before Task 8, which it does.
