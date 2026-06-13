# Side-bet ideas — for next time

Notes for refreshing the 20 side bets in a future edition. Nothing here is live yet;
this is a parking lot. To action: edit `DATA.sideBets` + the `COMPUTE`/leaderboard
logic in `dashboard.html`, and mirror the bet-unit list in `OPERATIONS.md`.

## Shortlisted to add (Leo's picks)

| New bet | Measures | Type | Data source | Replaces |
|---------|----------|------|-------------|----------|
| **The Entertainers** | Most goals in a team's matches (scored + conceded combined) | ∑ Tournament | recentScores (hs+as in their games) | a weak running-total bet |
| **Iron Curtain** | Fewest goals conceded | ∑ Tournament | recentScores (min goals against) | a weak running-total bet |
| **Early Bath** | Earliest red card of the tournament (lowest minute) | ⚡ Single game | red-card event minute | **Seeing Red** (most reds) — confirmed |
| **Cinderella** | Furthest-progressing team with the lowest FIFA rank | Tournament arc | FIFA rank (already in `ALLOC`) + round reached | a weak running-total bet |
| **Houdini** | Scrapes out of the group on fewest points / worst goal difference | Tournament arc | group standings (points, GD) | a weak running-total bet |

Confirmed replacement: **Early Bath → Seeing Red**.
Other four: map onto the weakest current bets next time (candidates to retire below).

## Candidate bets to retire (sparse / tie-prone / dull)

- **Seeing Red** (most red cards) — replaced by Early Bath.
- **Wrong Net** (most own goals) — rare, usually 0 or a multi-way tie.
- **Spot-Kick Magnet** (most penalties conceded) — sparse early, often tied.
- (Review **All Sizzle No Steak** / **Shooting Gallery** if more variety wanted.)

## Implementation notes

- **The Entertainers / Iron Curtain** — drop-in: same shape as the existing running
  totals (sum a per-team number across recentScores, highest/lowest wins). Iron Curtain
  is the inverse of The Sieve; The Wall already covers clean sheets, so these are distinct.
- **Early Bath** — single-game ⚡ award like Off Like a Rocket. Needs the *minute* of red
  cards captured per match (add a field e.g. `firstRed:{m,t}` or read from events). Lowest
  minute across the tournament wins.
- **Cinderella & Houdini** — these need a notion of **round reached / group outcome**, which
  the current data model does NOT track (it only holds group-stage results + remaining
  fixtures). Would need a new per-team field (e.g. stage reached, group points/GD) for the
  routine to maintain. More work than the others — flag for design before adopting.
- Co-owned teams (Japan, Curaçao) and the Champion's Bonus settle-logic apply unchanged.

## Fuller idea bank (not picked, kept for reference)

Drop-in verifiable from the ESPN feed: **The Flagged** (most offsides), **Safe Hands /
The Cat** (most GK saves), **Dead-Eye** (most shots on target), **Pass Masters** (most
completed passes / best accuracy), **The Enforcers** (most tackles), **Whipped In** (most
crosses), **Snoozefest FC** (fewest goals in their matches).

Single-game / moment: **Goal Glut** (highest-scoring match), **Ten Men Standing** (win/draw
after a red), **Booked in a Blink** (earliest yellow), **Shootout Slayer** (win a penalty
shootout), **Super Sub Special** (sub scores a 90'+ winner).

Tournament arc: **Perfect Group** (win all three group games), **Home Comforts** (best host
nation — USA/Canada/Mexico).

Pure banter (settle by group vote, not auto-verifiable): **The Wig** (worst haircut),
**Anthem Belter**, **VAR Pantomime**, **Manager Meltdown**, **Waterworks**.

> Data caveat: offsides/saves/passes/tackles/crosses are in ESPN's summary feed for a
> tournament this size, but coverage varies per match — check the field is populated before
> trusting any Tier-1 stat bet (same spirit as the "use the summary endpoint" rule).
