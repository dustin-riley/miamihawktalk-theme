# MiamiHawkTalk Workspace Restructure — Design

**Date:** 2026-07-19
**Status:** Approved, ready for planning

## Goal

Contain the three MiamiHawkTalk repositories under one parent folder, give each
a `CLAUDE.md` holding only what every agent in that repo needs, and merge the
scattered backlogs into one list.

## Problem

The three repositories sit loose among ~21 unrelated projects in
`~/Development/`, with nothing tying them together:

```
~/Development/
├── miamihawktalk-theme/            (has CLAUDE.md)
├── discourse-redhawks-schedule/    (no CLAUDE.md)
├── miamihawktalk-schedule/         (no CLAUDE.md)
└── ...18 unrelated projects
```

Two concrete symptoms:

- The theme repo's `docs/superpowers/` holds the athletics schedule spec and
  plan, which describe the *other two* repos entirely.
- Starting an agent anywhere means it has no way to know the other two repos
  exist, or which one owns a given change.

## Decisions

| Question | Decision |
|---|---|
| Parent folder | `~/Development/MiamiHawkTalk/`, a **minimal git repo** |
| What the parent tracks | `CLAUDE.md`, `BACKLOG.md`, `.gitignore`, and `docs/superpowers/` for workspace-level design docs — child repos gitignored |
| Parent `CLAUDE.md` scope | Routing, separation of responsibilities, durable shared facts. **No deploy mechanics** |
| Schedule spec + plan | Move to the **plugin** repo |
| Scope of new `CLAUDE.md` files | The three MiamiHawkTalk repos only |
| Backlog | One `BACKLOG.md` at the parent, merged |
| `discourse-styling` skill | Leave in the theme repo, untouched |

### Why the parent is a repo after all

The parent was first specified as a plain folder. Reading `docs/ROADMAP.md`
changed that: it is a curated 68-line document — five themed tracks, effort
ratings, a sequencing section, open questions — not a scratch list. Moving it
into an untracked folder would have dropped both its history and its only
backup. A minimal repo keeps the single-place convenience without that loss.

The parent tracks planning artifacts only. The three child repos stay fully
independent, with their own remotes and history, and are gitignored so the
parent never tries to track their contents.

## Two rules governing content

These came from the user and apply to every file this project writes.

**1. `CLAUDE.md` holds only what every agent touching that folder needs.**
Anything conditional — needed by some agents but not all — becomes a skill
instead. The theme repo already demonstrates this: its `discourse-styling`
skill carries the SCSS colour-scheme rules, and `CLAUDE.md` merely points at it.

**2. No todos in `CLAUDE.md`.** Backlog items live in `BACKLOG.md`. `CLAUDE.md`
may reference it but must not restate it.

## Target structure

```
~/Development/MiamiHawkTalk/              # minimal git repo
├── .gitignore                            # ignores the three child repos
├── CLAUDE.md                             # routing + shared facts, ~30 lines
├── BACKLOG.md                            # merged, version controlled
├── docs/superpowers/                     # workspace-level design docs only
│   ├── specs/2026-07-19-workspace-restructure-design.md
│   └── plans/2026-07-19-workspace-restructure.md
├── miamihawktalk-theme/                  # own repo, gitignored here
│   ├── CLAUDE.md                         # trimmed
│   └── .claude/skills/discourse-styling/ # untouched
├── discourse-redhawks-schedule/          # own repo, gitignored here
│   ├── CLAUDE.md                         # new
│   └── docs/superpowers/                 # schedule spec + plan land here
└── miamihawktalk-schedule/               # own repo, gitignored here
    └── CLAUDE.md                         # new
```

## File contents

### Parent `CLAUDE.md`

Its one job is routing: an agent started here must learn which repo owns the
change it was asked to make, and go there. Contents:

- What each repo is and what it owns, in one line each.
- The rule that changes are made **inside** the relevant child repo, whose own
  `CLAUDE.md` governs — the parent never duplicates a child's guidance.
- Durable shared facts: the site is `miamihawktalk.fans`, self-hosted Discourse,
  currently 2026.7.0-latest; this is an **unofficial fan site**, so Miami
  University trademarks are never used.
- A pointer to `BACKLOG.md`.

Explicitly **not** included: per-repo deploy mechanics, runtime architecture, or
anything restating a child `CLAUDE.md`. Deploy cost differs sharply between the
plugin and the two theme-side repos, and that belongs with each.

### Plugin `CLAUDE.md` (`discourse-redhawks-schedule`)

Every agent in this repo needs:

- What it is: a Discourse plugin that fetches the SIDEARM athletics RSS calendar
  server-side every 30 minutes and serves it at `/redhawks-schedule.json`. It
  exists because the vendor sends no CORS headers, so a browser fetch is
  impossible.
- **Deploying needs `./launcher rebuild app`** — minutes of real downtime. This
  is the single most important difference from the theme-side repos.
- **`rails runner` must run as the `discourse` user**
  (`sudo -E -u discourse bundle exec rails runner …`). Postgres uses peer
  authentication and rejects root. This cost a debugging cycle during deploy.
- Tests: `~/.gem/ruby/2.6.0/bin/rspec spec/lib/parser_spec.rb` — rspec is
  installed user-scoped and is **not** on `PATH`.
- **Target Ruby is 2.6.10** locally while the container runs 3.x, so `filter_map`
  and other 2.7+ syntax fail only on the dev machine. This has already bitten.
- Architecture: the parser is deliberately free of Rails, HTTP and persistence
  so it can be unit tested without a Discourse checkout. Keep it that way.
- A pointer to `NOTES-api-verification.md` and `docs/superpowers/`.

### Component `CLAUDE.md` (`miamihawktalk-schedule`)

Every agent in this repo needs:

- What it is: the theme component rendering the sidebar block, reading
  `/redhawks-schedule.json` from the plugin.
- Deploys by git push plus **Update** in Discourse admin, then a hard refresh.
  No rebuild, no downtime — which is *why* it is separate from the plugin.
- **Translations must go through `themePrefix`.** Theme components store strings
  as `theme_translation.<theme_id>.<key>`; keys sit directly under `en:` with no
  `js:` level, and a bare `i18n("key")` renders a literal `[en.key]` placeholder.
  This shipped as a Critical bug and was caught only in final review.
- It renders into the `after-sidebar-sections` plugin outlet.
- No build step, no package manager, no test runner.
- It must render **nothing** — no error text, no empty container — when there is
  no data. Zero upcoming events is the normal state every June.
- Colours come from the parent theme; never hardcode one.

### Theme `CLAUDE.md` (`miamihawktalk-theme`)

Keep as-is apart from:

- Replace the `docs/ROADMAP.md` bullet under "Project docs" with a pointer to
  the parent `BACKLOG.md`.
- The "Phase 1 is deliberately near-zero JavaScript" constraint is guidance, not
  a todo, and **stays** — it explains why this repo has no `.gjs`.
- Note that `docs/superpowers/` now holds only theme-related specs, the schedule
  documents having moved to the plugin repo.

### `BACKLOG.md`

Merges `docs/ROADMAP.md` with the open items from the schedule build:

- The five existing tracks, carried over intact.
- Track D's "Schedule & record widget" marked **substantially delivered**, with
  what remains named explicitly: record/W-L display and next-game countdown.
- Deferred component work: the stale-tab re-filter (a tab left open all day
  keeps showing games that already started) and the `(Exhibition)` suffix
  truncating the opponent name.
- Unfinished live verification: the TBA row checked against the athletics site,
  dark scheme, logged-out, and mobile.

Items are tagged with the repo they belong to so the parent list stays routable.

## Sequencing

Order matters in two places.

**The theme repo moves last.** The running session's working directory is
`~/Development/miamihawktalk-theme`; moving it out from under an active session
breaks subsequent commands.

**`ROADMAP.md` is deleted in the same commit that its merged content lands** in
`BACKLOG.md`, so the content is never in limbo.

1. Create `~/Development/MiamiHawkTalk/` and `git init` it; write `.gitignore`.
2. Move `discourse-redhawks-schedule/` and `miamihawktalk-schedule/` in.
3. Write both new `CLAUDE.md` files.
4. Move the **athletics schedule** documents (`2026-07-18-athletics-schedule-*`)
   into the plugin repo, fixing the stale absolute paths the plan hardcodes
   throughout. Move **this restructure's** own spec and plan
   (`2026-07-19-workspace-restructure-*`) into the parent repo, since they
   describe the workspace rather than any one repo.
5. Write `BACKLOG.md`; trim the theme `CLAUDE.md`; delete `docs/ROADMAP.md`.
6. Move `miamihawktalk-theme/` in last.
7. Restart Claude in `~/Development/MiamiHawkTalk/`.

## Verification

- Each child repo still reports its own remote and unbroken history after the
  move (`git -C <repo> log --oneline -1` and `git -C <repo> remote -v`).
- `git status` in the parent shows only `CLAUDE.md`, `BACKLOG.md` and
  `.gitignore` — never child repo contents.
- No file under the parent contains a `~/Development/<repo>` path that skips the
  `MiamiHawkTalk/` level.
- The real test is step 7: starting an agent in the parent and asking for a
  change to, say, the sidebar row layout should send it to the component repo
  without further prompting.

## Out of scope

- `CLAUDE.md` files for the other ~18 projects in `~/Development/`.
- Moving or changing the `discourse-styling` skill.
- Any change to deployed code. This restructure touches local layout and
  documentation only; Discourse clones from GitHub, so the live site is
  unaffected throughout.
- Pushing the parent repo to GitHub — worth doing for backup, but it is an
  outward-facing action to confirm separately.
