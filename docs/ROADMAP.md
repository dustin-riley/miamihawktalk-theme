# MiamiHawkTalk — Feature Roadmap / Idea Backlog

A running to-do list of ideas for the site. Grouped by track and tagged with
rough effort + whether an idea needs an external data feed. Nothing here is
committed scope yet — it's the menu we pick from.

**Tags:** `[theme]` styling only · `[glimmer]` needs custom JS components ·
`[feature]` meaningful build · `[automation]` scheduled/AI job I can run ·
`[data]` needs an external feed (e.g. ESPN scores/schedule).
**Effort:** 🟢 small · 🟡 medium · 🔴 large.

---

## ✅ Phase 1 — Branded theme (DONE)
- [x] Full theme: light/dark RedHawks color schemes
- [x] Original wordmark + monogram + favicon
- [x] Inter typography, accent discipline, header polish
- [x] Installed at github.com/dustin-riley/miamihawktalk-theme
- [ ] Final sign-off: confirm dark-mode legibility + set as site default

---

## 🏟️ Track A — Game-day energy
- [ ] **Game-day mode** — auto banner on game days (matchup, kickoff, TV, live-thread link); hidden off-days. `[theme]`+`[automation]` 🟡 (static/hand-set version 🟢; auto-detecting needs `[data]`)
- [ ] **Auto-created game threads** — scheduled agent opens a pinned "Game Thread: Miami vs X" before each game. `[automation]` 🟡
- [ ] **Live score bug** — small scoreboard in header/sidebar during games. `[glimmer]`+`[data]` 🔴
- [ ] **Post-game "Grade the RedHawks"** — auto-posted poll after each game, results roll up over the season. `[automation]`+`[feature]` 🟡

## 🎯 Track B — Fandom & competition
- [ ] **Season-long Pick'em + leaderboard** — weekly predictions, points, standings. Stickiest daily-return driver. `[feature]`+`[data]` 🔴
- [ ] **MAC Tournament / March bracket challenge** — basketball bracket pool. `[feature]` 🟡
- [ ] **Prediction-cred badges** — flair for accurate callers; builds on Pick'em. `[feature]` 🟡

## 🦅 Track C — Miami heritage (unique to this school)
- [ ] **"Cradle of Coaches" touches** — rotating quote / heritage strip / about page. `[theme]` 🟢
- [ ] **"This Day in RedHawks History"** — automated daily post or sidebar snippet. `[automation]` 🟡 (needs a content source)
- [ ] **Rivalry-week takeover** — Victory Bell (vs Cincinnati) / MAC rivalry banner + countdown. `[theme]`+`[automation]` 🟡
- [ ] **"Love and Honor" welcome ritual** — branded new-member greeting. `[theme]` 🟢

## 🏠 Track D — A real front door (Phase 2 core)
- [ ] **Custom homepage** — featured hero, schedule/record strip, trending, recruiting headlines. `[glimmer]` 🔴
- [ ] **Schedule & record widget** — sidebar season schedule + W/L + next-game countdown. `[glimmer]`+`[data]` 🟡
- [ ] **Recruiting / commit tracker** — structured board for commits & targets. `[feature]` 🔴

## 🤖 Track E — AI-powered
- [ ] **"Catch me up"** — on-demand summary of a long game thread. `[automation]` 🟡
- [ ] **Weekly "Week Ahead" digest** — auto-posted every Monday. `[automation]` 🟢
- [ ] **Auto game recaps** — generated thread after each game. `[automation]`+`[data]` 🟡

---

## Suggested sequencing
1. **Close Phase 1** (dark-mode check + set default).
2. **Phase 2 = Track D custom homepage** — the biggest visual leap; makes the
   site feel like a destination. Naturally pairs with the schedule/record widget.
3. **First automation win = Track E "Week Ahead" digest** — 🟢 effort, high
   visibility, proves out the scheduled-agent pattern before bigger builds.
4. **Signature feature = Track A game-day mode** — start with the hand-set
   version, add the schedule feed later.
5. **Big engagement bet = Track B Pick'em** — most work, most payoff; do once
   the foundations above are in.

## Open questions to resolve before the data-dependent items
- Is there a reliable, allowed feed for Miami (OH) schedule + scores? (ESPN's
  undocumented API is the likely candidate — needs validation.)
- Which sports are in scope for game-day/Pick'em (football + men's basketball
  first, or all)?
- Any existing plugins installed (polls, etc.) we can build on?
