# WC 2026 Sweepstake

A live, auto-updating dashboard for a 10-person 2026 FIFA World Cup sweepstake.

- **Live board:** https://leo-gecco.github.io/wc-sweepstake/ (GitHub Pages serves the `main` branch)
- **Pot:** £200 — 10 players × £20. The whole pot rides on side bets (no entry-fee refunds).
- **Draw:** all 48 World Cup teams were drawn live (screen-recorded). Each player got one top-10
  team plus others; two players drew a bonus 5th team that is **shared** with its original owner.

## How it works (the short version)

`dashboard.html` is a single self-contained page. It holds the raw data in a `DATA` block and the
draw in an `ALLOC` array, and **recomputes everything (side-bet leaders, money standings, the
leaderboard-over-time chart, the tracker) in the browser on every page load**. There is no database
and no build step — to change what the board shows, you edit the data in `dashboard.html` and push to
`main`; GitHub Pages redeploys within ~a minute.

A scheduled **Claude Code routine** does this automatically each day (see `OPERATIONS.md`):
it pulls completed match stats from ESPN's public API, prepends them to `recentScores`, trims the
played game out of `fixtures`, updates the timestamp, and pushes to `main`. It only ever writes
**full-time** results (never mid-game). It also writes a paste-ready WhatsApp digest twice a week
(Monday & Friday, 08:15 UK), covering everything since the last one, into `whatsapp/`.

## Files

| File | What it is |
|------|------------|
| `dashboard.html` | The live board. Self-contained HTML/CSS/JS. **Only the `DATA` block and `ALLOC` array carry data** — everything else is rendering/logic. |
| `index.html` | Tiny redirect to `dashboard.html` (so the bare Pages URL works). |
| `OPERATIONS.md` | Full instructions for the daily updater routine: the exact prompt to paste into Claude Code, the ESPN data flow, the schedule/cron, the push method, the WhatsApp format, and all the edge-case rules. Start here to run or modify the automation. |
| `whatsapp/{YYYY-MM-DD}.txt` | The daily group-chat message the routine generates. One file per day. |

> Two more tools live in the owner's local folder (not in this repo) and were used once before the
> tournament: `draw.html` (the live team draw) and `draw-5th-team.html` (the bonus-team draw). They are
> not needed to run the sweepstake going forward.

## Data model (inside `dashboard.html`)

- `DATA.lastUpdated` — timestamp shown in the header chip, e.g. `"12 Jun 2026, 08:00 BST"`.
- `DATA.pot` — total pot (200).
- `DATA.sideBets` — the 20 side bets (name + description). Leaders/values are computed, not stored.
- `DATA.recentScores` — array of completed matches, newest first. Each row has ~27 fields (score,
  cards, corners, fouls, possession, shots, a `venue` "CITY, COUNTRY", plus moment markers `fg`/`lw`/`cb`).
  This is the only thing that grows during the tournament.
- `KNOCKOUTS` — the 32 knockout ties (R32→Final) shown below the group fixtures: each has an ESPN `id`,
  round, date, time, `venue`, and slot labels (e.g. `"Winner A"`, `"3rd C/E/F/H/I"`, `"Winner R32-1"`).
  The routine fills in real team names + scores as rounds resolve. Display-only — does not feed side bets.
- `DATA.fixtures` — the remaining group-stage fixtures with UK (BST) kick-off times. Played games are
  removed as results come in (the board also hides any fixture whose teams already appear in results).
  Each fixture also carries a `venue` ("CITY, COUNTRY", e.g. `"KANSAS CITY, USA"`) shown on the fixture card; it was sourced from ESPN's scoreboard (`competition.venue.address`).
- `ALLOC` — the draw: `["Player",[["Team","Group",FIFArank], ...]]`. **Ownership is derived from this
  at runtime** — change a name here and the whole board follows.

### Side bets (20, £10 each)

14 build up over the whole tournament; 6 are decided by a single standout game (badged on the cards as
`∑ Tournament` vs `⚡ Single game`):

Goal Machine (most goals), The Sieve (most conceded), Card Sharks (yellows), Seeing Red (reds),
Wrong Net (own goals), Spot-Kick Magnet (penalties conceded), Corner Shop (corners), The Hatchet Men
(fouls), The Wall (clean sheets), Aerial Threat (headed goals), Super Sub (sub goals), Tiki-Taka
(avg possession), Shooting Gallery (shots faced), All Sizzle No Steak (shots per goal) — plus the
single-game ones: The Hammering (heaviest defeat), Off Like a Rocket (fastest goal), Fergie Time
(latest 90'+ winner), Comeback Kings (biggest comeback), Smash & Grab (won on lowest possession),
Total Domination (highest possession in a match).

When a single team leads a bet its owner banks £10 (co-owned teams split it £5/£5). If two or more punters are tied for a bet — or a bet has no winner yet — its £10 is not split: it rolls into the **Champion's Bonus**, won at the end by whoever owns the tournament-winning team. The board has a **Rules** tab explaining this.

### Co-owned teams (important quirk)

Two teams are owned by two players each (a shared bonus pick), so their bet winnings split:
**Japan = Leo & Tom**, **Curaçao = Jamie & Brownout**. The render and standings handle this
automatically via the `ALLOC` array; `OPERATIONS.md` reminds the routine to credit both owners.

## Running / changing it

- **Automated daily updates:** follow `OPERATIONS.md` — paste its prompt into a Claude Code routine,
  set the cron, and it self-publishes. The write token is configured in that prompt (it is not stored
  in this repo).
- **Manual edit / redeploy:** edit the `DATA` block (or `ALLOC`) in `dashboard.html`, commit to `main`,
  and Pages redeploys. The board re-renders from the data on load — nothing else to touch.

## Status / open ideas

Tournament is underway (kicked off 11 June 2026). The **Champion's Bonus** is live: any contested or
no-winner side bet rolls its £10 into a pot won by the owner of the tournament-winning team. Set
`DATA.champion` to the winning team's name at the end and the board awards the bonus automatically.
