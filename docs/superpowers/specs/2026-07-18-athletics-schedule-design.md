# Upcoming Games — Design

**Date:** 2026-07-18
**Status:** Approved, ready for planning

## Goal

Show Miami University Athletics' upcoming events in the miamihawktalk.fans
sidebar, sourced from the athletics department's public RSS calendar.

## Source data

`https://miamiredhawks.com/calendar.ashx/calendar.rss` — a SIDEARM Sports feed.
A snapshot taken 2026-07-18 (`~144KB`) held 141 `<item>` elements covering the
full athletic year across every sport.

Each item is well structured:

```xml
<item>
  <title>8/16 7:00 PM Miami University Women's Soccer at Xavier</title>
  <link>https://miamiredhawks.com/calendar.aspx?game_id=20926&amp;sport_id=16</link>
  <ev:location>Cincinnati, Ohio</ev:location>
  <ev:startdate>2026-08-16T23:00:00.0000000Z</ev:startdate>
  <s:localstartdate>2026-08-16T19:00:00.0000000</s:localstartdate>
  <s:opponentlogo>https://miamiredhawks.com/images/logos/Xavier_.png</s:opponentlogo>
  <s:opponent>Xavier</s:opponent>
  <s:gameid>20926</s:gameid>
</item>
```

Observed characteristics of the snapshot, which the parser must not assume hold
forever:

- No `TBA`/`TBD` times and no midnight-placeholder start times.
- 66 of 141 titles use ` at ` (away); the rest use ` vs `.
- Multi-day events appear as repeated items sharing a title: `Men's Golf vs MAC
  Championship- Talis Park` appears 3 times, `Hockey vs NCHC First Round
  (Best-of-3)` 3 times.

## Decisions

| Question | Decision |
|---|---|
| Placement | Sidebar block |
| Sports included | All of them, chronological |
| Data delivery | Discourse plugin, server-side fetch |
| Packaging | Plugin for data, theme component for UI |
| Row design | Opponent logo + vs/at + opponent, sport and date beneath |
| Multi-day events | Collapsed into one row with a date range |
| Timezone | Visitor's local timezone |

### Why a plugin rather than a browser fetch

A theme component runs in the visitor's browser, and `miamiredhawks.com` does
not send CORS headers, so a direct `fetch()` is blocked. A Cloudflare Worker
proxy was considered and rejected once self-hosting was confirmed: a plugin
fetches server-side, which removes the CORS problem, the third-party service,
and the account to maintain.

### Why the plugin/component split

Plugins require `./launcher rebuild app` to update and can fail a site's boot;
theme components update with a git pull and a browser refresh and cannot. So the
backend — small, stable, essentially never changing after it works — is the
plugin, and everything that will be iterated on (layout, CSS, copy, settings)
stays in a theme component.

## Architecture

```
miamiredhawks.com/calendar.ashx/calendar.rss
        |  Sidekiq, every 30 minutes, server-side
[1] discourse-redhawks-schedule  (plugin)
        |  PluginStore -> GET /redhawks-schedule.json  (same origin)
[2] miamihawktalk-schedule  (theme component)
        |
[3] "Upcoming Games" sidebar section
```

## Component 1 — `discourse-redhawks-schedule` (plugin)

```
plugin.rb
config/settings.yml
app/jobs/scheduled/fetch_redhawks_schedule.rb
app/controllers/redhawks_schedule_controller.rb
lib/redhawks_schedule/parser.rb
spec/lib/parser_spec.rb
spec/fixtures/calendar.rss
```

### Parser (`lib/redhawks_schedule/parser.rb`)

A plain Ruby class: XML string in, array of event hashes out. No Rails, no HTTP,
no persistence — this is the unit that carries all the interesting logic and all
the tests.

Uses Nokogiri (already bundled with Discourse) with namespace-aware selectors
rather than string matching.

Per item:

- **sport** — strip the leading `M/D H:MM AM/PM ` and the `Miami University `
  prefix from the title, then take everything before ` vs ` or ` at `.
- **opponent** — `s:opponent`, falling back to the title remainder after the
  ` vs ` / ` at ` separator.
- **homeAway** — `at` in the title means away, `vs` means home. Neutral-site
  games are published as `vs` and are not distinguishable from true home games;
  they are treated as home.
- **startUtc / endUtc** — `ev:startdate` / `ev:enddate`, ISO 8601 UTC.
- **location** — `ev:location`.
- **opponentLogo** — `s:opponentlogo`.
- **url** — `link`.
- **id** — `s:gameid`.

Then, in order:

1. Drop events whose `startUtc` is in the past.
2. Sort ascending by `startUtc`.
3. Collapse runs of same-sport, same-title events into one entry whose
   `startUtc` is the first item's and whose `endUtc` is the last item's.

Times that arrive as `TBA`/`TBD` or with a midnight placeholder are absent from
the current feed but must not raise; they are emitted with a null time and
rendered as `TBA`.

### Job (`app/jobs/scheduled/fetch_redhawks_schedule.rb`)

`Jobs::Scheduled`, `every 30.minutes`. Fetches via `FinalDestination::HTTP`,
passes the body to the parser, and writes the result plus a `fetched_at` stamp
to `PluginStore`.

**A failed fetch or a parse error leaves the previous value in place.** This is
the entire stale-if-error strategy: an outage at miamiredhawks.com is invisible
to forum users until the data ages out naturally. Failures are logged, not
raised.

`PluginStore` is used deliberately in place of a custom table — the payload is a
few KB and needs no querying, so a migration would be overhead with no benefit.

### Controller

`GET /redhawks-schedule.json` returns `{ generated_at, events: [...] }`.

- `requires_login false` — anonymous visitors must get the sidebar too.
- `skip_before_action :check_xhr`.
- `Cache-Control: public, max-age=900`.
- Returns an empty `events` array, not an error, when the store is empty.

### Site settings

- `redhawks_schedule_enabled` — bool, default true
- `redhawks_schedule_feed_url` — string, default the calendar.ashx URL above

## Component 2 — `miamihawktalk-schedule` (theme component)

```
about.json          # component: true
settings.yml
javascripts/discourse/api-initializers/schedule.js
javascripts/discourse/components/upcoming-games.gjs
common/common.scss
locales/en.yml
```

Kept separate from the main `miamihawktalk-theme` repo so it can be disabled or
updated independently, and so the theme stays SCSS-only. It inherits the theme's
color variables (`$tertiary`, `$primary`, `$primary-medium`, `$primary-low`)
automatically and must not redefine the palette.

### Rendering

A Glimmer component rendered into a sidebar plugin outlet.

**Open item for the first implementation step:** Discourse's `addSidebarSection`
API is organised around link rows and will not produce the two-line, logo-
bearing layout below. The rich layout needs a plugin outlet
(`below-sidebar-sections` or equivalent), whose exact name must be confirmed
against the running Discourse version before building. If no suitable outlet
exists on that version, the fallback is the text-only single-line layout via the
standard section API — that is a visible change to an approved design and
requires checking back before proceeding.

### Row

```
[logo]  vs Xavier
        W SOCCER - Aug 16, 7:00p
```

- Logo 24px from `opponentLogo`; on error the `img` is hidden and text reflows.
  Some of these URLs will eventually 404.
- `vs` / `at` in `$tertiary`; opponent name in `$primary`.
- Sport label uppercase, small, `$primary-medium`, shortened via a lookup map
  (`Women's Soccer` -> `W SOCCER`, `Men's Basketball` -> `M BASKETBALL`) so rows
  do not wrap at sidebar width (~200px).
- Times rendered in the visitor's local timezone with `Intl.DateTimeFormat`, no
  timezone label, since it is the reader's own clock.
- Collapsed multi-day events show a range: `Apr 27-29`.
- The whole row links to the event's `calendar.aspx` page, new tab.
- Footer link **See full schedule ->** to `https://miamiredhawks.com/calendar.aspx`.

### Data fetch

`GET /redhawks-schedule.json` on page load, cached in `localStorage` with a
30-minute TTL so navigation does not re-request.

### Settings

- `games_shown` — int, default 5, range 1-10
- `enabled_on_mobile` — bool, default true

### Empty and error states

If the request fails, returns malformed data, or returns zero upcoming events,
**the section renders nothing at all** — no error text, no empty container. Zero
events is the normal state every June; a persistent "Error loading games" box in
the sidebar for six weeks would be worse than absence.

## Verification

Per this project's rule that only the running site confirms rendering:

**Offline, before deploy:**

- `bundle exec rspec` against the parser, using the 2026-07-18 snapshot as a
  fixture. Covers: sport extraction across all title shapes, vs/at home/away,
  multi-day collapsing (the 3-item golf and hockey cases specifically),
  past-event filtering, and malformed/TBA input not raising.
- `python3 -m json.tool about.json`, `ruby -ryaml -e "YAML.load_file(...)"` on
  locales, per the existing repo checks.

**On the live site:**

- Sidebar renders in both MiamiHawkTalk Light and Dark.
- Desktop and mobile.
- Logged out, to confirm the anonymous path works.
- A row with a deliberately broken logo URL to confirm the fallback.
- Empty state, by pointing the feed setting at a stub that yields no events.

## Out of scope

- A dedicated `/games` route or full schedule page.
- Per-user sport filtering.
- Scores, results, or standings — this is upcoming events only.
- Game-thread creation or any link between events and topics.
