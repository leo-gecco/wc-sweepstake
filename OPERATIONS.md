> NOTE: This is the committed (repo-safe) copy of the routine instructions. The real write token is
> NOT in this file — it lives only in the Claude Code routine prompt (shown here as `<WRITE_TOKEN>`).
> Treat this repo as public.

# WC2026 Sweepstake — Claude Code Routine (cloud, laptop-off)

This sets up an automatic schedule-driven updater using **Claude Code Routines**, which run in
Anthropic's cloud on a schedule (no laptop needed).

---

## 1. Create the routine

1. Open Claude Code on the web → **Routines** → **New routine**.
2. **Repository:** connect `leo-gecco/wc-sweepstake`.
3. **Schedule:** Run **every 2 hours through the evening, then a single morning refresh** — cron (UTC):
   `0 7,17-23/2 * * *`. That fires at **18:00, 20:00, 22:00 and 00:00 BST** (every 2 hours from the earliest
   kickoff through midnight), then nothing overnight until a single **08:00 BST** refresh. Games that finish
   after midnight (kickoffs run as late as 05:00 BST, ending ~07:00) simply wait for the 08:00 run, which
   also recaps the whole night and writes the WhatsApp message (MESSAGE GATE). Most slots find nothing new
   and make no commit, so it's cheap. (If your routine reads cron in UK/BST rather than UTC, use
   `0 0,8,18-22/2 * * *` instead. The message gate keys off UK wall-clock either way.) Note: the **08:00 BST
   run is the single morning-message slot** — if you want a retry in case it fails, add a 10:00 BST run:
   `0 7,9,17-23/2 * * *`.
4. **Push permissions:** allow the routine to push to `main` (GitHub Pages serves `main`). If the
   routine is locked to `claude/…` branches, instead point GitHub Pages at that branch, or add an
   auto-merge action.
5. Paste the **Prompt** below.
6. Enable it for the tournament window (11 Jun – 19 Jul 2026); the prompt safely no-ops outside it.

---

## 2. The routine prompt (paste this in)

```
You are the daily updater for a live World Cup 2026 sweepstake dashboard. Each run starts fresh.

# REPO & FILE
- Repo: leo-gecco/wc-sweepstake (connected).
- Edit ONLY: dashboard.html
- Live site: https://leo-gecco.github.io/wc-sweepstake/ (GitHub Pages serves main).
- After editing, write the changed file(s) to the main branch so Pages redeploys — see PUSH METHOD.

# PUSH METHOD (important)
- Push with GIT over HTTPS, token INLINE in the URL. This is the only method used — do NOT use the GitHub
  connector (push_files / create_or_update_file): it requires re-uploading the whole file each run, which
  is cumbersome. git lets you commit incremental edits normally.
    git add -A
    git commit -m "<message>"
    git push "https://x-access-token:<WRITE_TOKEN>@github.com/leo-gecco/wc-sweepstake.git" HEAD:main
- VERIFY it landed every run: re-read dashboard.html from `main` (or check the latest commit) and confirm
  your new row + lastUpdated are actually there. Only report success when the change is live on `main`.
- If git push 403s ("permission denied to leo-gecco" / "git proxy token has no push scope"), the container
  is forcing its own read-only git proxy that overrides the inline token — STOP, report it plainly (do not
  claim success), and we'll switch to the GitHub Actions method below.

# STOP CONDITIONS
- Only act on matches dated 2026-06-11 to 2026-07-19 (UTC).
- If today is outside that window: do nothing, report "Outside tournament window."
- If no new completed matches are found: do nothing, report "No new matches."

# DATA SOURCE — ESPN public API (no key, structured JSON). TWO STEPS.
- The World Cup league slug is fifa.world. (Warm-up friendlies are under fifa.friendly — a different slug.)

STEP 1 — find the day's games (get event IDs):
  https://site.api.espn.com/apis/site/v2/sports/soccer/fifa.world/scoreboard?dates=YYYYMMDD  (UTC, no dashes)
  Use the scoreboard ONLY to list each match's event id + the two teams. Fetch yesterday AND today (UTC).

STEP 2 — get the real, completed stats (per event id):
  https://site.api.espn.com/apis/site/v2/sports/soccer/fifa.world/summary?event={id}
  IMPORTANT: the scoreboard often serves a STALE pre-match snapshot (status "Scheduled", 0-0, empty
  statistics) even AFTER a game has finished. Do NOT trust the scoreboard's score/stats/status. Always
  open the summary endpoint for each event — it has the authoritative final score, the boxscore team
  statistics, and the key events. (Verified on the real England 3-0 Costa Rica game: scoreboard showed
  "Scheduled / 0-0 / empty"; the summary had possession 81.3 vs 18.7, 28 vs 1 shots, 11 vs 1 corners, etc.)
  ONLY-WHEN-FINISHED (important — do not update mid-game): add a match ONLY when it is FULL-TIME. Check the
  status in the summary — `status.type.state` must be `"post"` (equivalently `status.type.completed == true`,
  i.e. FT / Full Time). If a game is IN PROGRESS (`state":"in"` — kicked off, half-time, or playing) or NOT
  STARTED (`"pre"`), SKIP it entirely this run: write NO row for it, do NOT move it out of `fixtures:`, and
  do NOT let it affect lastUpdated. It will be picked up on a later run once it has actually finished.
  Never write a partial or in-play row — a half-finished scoreline would corrupt the side-bet totals.
- Do NOT use BBC or other sources; ESPN is authoritative here.

# EXTRACT PER COMPLETED MATCH (from the SUMMARY endpoint)
- Final score + which side is home/away: from the summary header/competitors.
- Team statistics (boxscore): each team's statistics[] entries look like
  {"name":"possessionPct","displayValue":"81.3","label":"Possession"} — read possessionPct, wonCorners,
  foulsCommitted, totalShots, shotsOnTarget by their "name" and take "displayValue". (The scoreboard uses
  the same field names if it is ever populated, so the field mapping is identical either way.)
- Match events (goals/cards): from the summary's key-events / commentary list — each has the team,
  the minute (clock), a type/text (e.g. "Goal - Header", "Yellow Card", "Penalty"), penalty and own-goal
  flags, and the scorer with a position ("SUB" for substitutes). Use these for cards, own goals,
  penalties, headed goals, sub goals, the fastest goal, and the late/comeback markers.

Build these fields (h = home, a = away):
- hs/as  = competitor.score
- yh/ya  = count of yellowCard==true per team (a 2nd yellow = 1 yellow + 1 red; ESPN flags both)
- rh/ra  = count of redCard==true per team
- oh/oa  = count of ownGoal==true per team
- ph/pa  = penalties conceded = count of penaltyKick==true awarded to the OPPOSING team
- ch/ca  = wonCorners
- foh/foa = foulsCommitted
- poh/poa = possessionPct (round to integer; they should sum to ~100)
- shh/sha = totalShots
- hgh/hga = count of goals whose type.text contains "Header"
- sgh/sga = count of goals where the scorer's athletesInvolved[].position == "SUB"
- fg = { m: earliest goal minute in the match, t: "scoring team name" }  (omit for 0-0)
- lw = { m, t } ONLY if a goal at minute >= 90 won or rescued the game for that team; else omit
- cb = { t, d } ONLY if a team won/drew after being >= 2 goals down (read the goal order); else omit

# TEAM NAME NORMALISATION (ESPN -> our names)
Czechia->Czech Republic, USA->United States, Korea Republic->South Korea, Turkiye/Türkiye->Turkey,
IR Iran->Iran, Cote d'Ivoire->Ivory Coast, Cabo Verde->Cape Verde, Curacao/Curaçao->Curacao,
Bosnia & Herzegovina->Bosnia-Herzegovina, DR Congo->DR Congo.
If a team still doesn't match the ALLOC list in dashboard.html, add the match anyway and flag it.

# GROUP LOOKUP (team -> group) — CONFIRMED FINAL DRAW, use this map exactly:
A: Mexico, South Africa, South Korea, Czech Republic
B: Canada, Bosnia-Herzegovina, Qatar, Switzerland
C: Brazil, Morocco, Haiti, Scotland
D: United States, Paraguay, Australia, Turkey
E: Germany, Curacao, Ivory Coast, Ecuador
F: Netherlands, Japan, Sweden, Tunisia
G: Belgium, Egypt, Iran, New Zealand
H: Spain, Cape Verde, Saudi Arabia, Uruguay
I: France, Senegal, Iraq, Norway
J: Argentina, Algeria, Austria, Jordan
K: Portugal, DR Congo, Uzbekistan, Colombia
L: England, Croatia, Ghana, Panama
(Write it as "Group A" etc. in the row. These match the ALLOC + fixtures already in dashboard.html.)

# ROW FORMAT (prepend to the recentScores array, newest first)
{ date:"DD Mmm 2026", group:"Group X", home:"Team", hs:0, away:"Team", as:0, yh:0, ya:0, rh:0, ra:0, oh:0, oa:0, ph:0, pa:0, ch:0, ca:0, foh:0, foa:0, poh:0, poa:0, hgh:0, hga:0, sgh:0, sga:0, shh:0, sha:0, fg:{m:0,t:"Team"} },
(include lw and cb only when they occurred.)
- `date:` = the UK (Europe/London) kick-off date, so results sit on the same day as the UK-dated fixtures
  (a late US game that finishes after UK midnight belongs to the next UK day).

# PROCEDURE (board refreshes on the schedule above; message once a day at 08:00)
1. Read dashboard.html; find `recentScores: [` and `fixtures: [`.

2. EVERY RUN — refresh the board:
   a. Pull completed matches via the two-step ESPN flow above, for yesterday AND today (UTC).
   b. IDEMPOTENCY: skip any match whose date+home+away row already exists. Never duplicate.
   c. Prepend new rows to recentScores (newest UK date first; order by group letter within a date).
   d. Move any now-played matches out of the `fixtures:` array.
   e. Update `lastUpdated:` to the current UK date+time, "DD Mmm YYYY, HH:MM BST" (Europe/London).
   f. VALIDATE the file parses (all 26 fields per row, numbers non-negative ints, strings quoted,
      brackets/commas intact). If it fails: do NOT commit, report the problem, stop.
   g. If anything changed, commit dashboard.html to `main` per PUSH METHOD above, message like
      "Board update: {UK datetime}". (If there were no new matches, nothing changed — skip the commit and
      continue to step 3.)

3. MESSAGE GATE — only the 08:00 run writes the WhatsApp message. Write it this run only if BOTH:
   - the current UK (Europe/London) time is 08:00 or later, AND
   - `whatsapp/{today-UK-date}.txt` does NOT already exist in the repo.
   Otherwise skip the message entirely and just report "board updated; message not due / already sent".
   This yields exactly ONE message per UK day — the first run at/after 08:00 — independent of the earlier
   board refreshes.

4. Build the morning message (recap + preview) covering the LATEST DAY'S full progress + today's games —
   NOT just what changed since the previous run. "Latest day" = every game completed since yesterday
   morning: rows dated YESTERDAY (UK) plus any overnight games dated TODAY (UK) that have finished. Call
   that set R.
   - BEFORE/AFTER without memory: compute the £ standings across all 20 side bets TWICE —
     once from ALL recentScores (AFTER), once EXCLUDING set R (BEFORE). The diff is exactly what those
     games changed (leaders won/lost, who climbed the money table).
   - Content: recap the results in R, the movers from that diff, the current money standings, then a
     preview of TODAY's fixtures (from `fixtures:`, today's UK date, with their BST kickoff times).
   - Write `whatsapp/{today-UK-date}.txt` and commit it to `main` per PUSH METHOD, message
     "Daily message: {date}". If dashboard.html also changed this run, push both in one commit.

5. Report: matches added (with scores), whether the daily message was sent this run, side-bet/standings
   changes, manual-review flags, push status. If the message WAS sent, paste it verbatim for copy-paste.

# HOW TO COMPUTE STANDINGS (for BEFORE/AFTER)
For ALL 20 side bets — the 14 running totals (most goals scored, most conceded, most yellows, reds, own
goals, penalties conceded, corners, fouls, clean sheets, headed goals, sub goals, highest avg possession,
most shots faced, most shots per goal) plus the 6 single-game awards (heaviest defeat, fastest goal,
latest 90'+ winner, biggest comeback, lowest winning possession, highest single-game possession): find the leading
team(s) from recentScores, map each to its punter(s) via the ALLOC list in dashboard.html, and award £10
per bet. Split the £10 equally across the tied teams, then split each team's share equally across that
team's owners.
CO-OWNED TEAMS — two teams appear under TWO punters in ALLOC: **Japan = Leo & Tom**, **Curacao = Jamie &
Brownout**. If a co-owned team leads or ties a bet, both owners share that team's money. Whenever you map a
team to a punter (standings or the WhatsApp message), check for a second owner and name/credit both.
Sum per punter to get the £ standings across all 20 bets — this matches the dashboard's Leaderboard Over Time, which now counts every side bet.

# WHATSAPP SUMMARY (paste-ready, entertaining) — the single daily 08:00 message
A recap of the games completed since yesterday's message (yesterday's results + any overnight games), the
money swings they caused (with metrics), the current standings, then a preview of TODAY's games. It reads
as a "good morning, here's where we're at + what's on today" brief.
For the preview, read TODAY's fixtures from the `fixtures:` array in dashboard.html (filter to today's UK
date) — each entry already carries its UK (BST) kickoff `time`, so no timezone maths is needed. Map each
team to its punter(s) via ALLOC (name both owners for the co-owned teams) so you can say who has skin in
each game and what's at stake.

Rules:
- WhatsApp formatting: wrap text in *single asterisks* to BOLD it (WhatsApp renders *text* as bold). Bold
  the title line, EVERY section header, and a punter's name+total when they move. Don't bold whole
  sentences. Emojis welcome.
- Readability first: a blank line between every section, and each result / mover / fixture on its OWN
  short line. It's read on a phone — no walls of text.
- ALWAYS put the country's flag emoji immediately before EVERY country/team name, everywhere in the
  message (results, movers, money table, fixtures, one-to-watch). Never write a bare country name. Use
  the FLAG map already defined in dashboard.html for the exact emoji; for any country not in it, use that
  nation's standard flag emoji.
- British English, proper group-chat banter: funny, a bit savage, never nasty. Sentences under 20 words.
  No em dashes. Ribbing for flops, hype for cash-ins. Give the bottom punter a cheeky dig.
- EVERY side-bet line must quote the actual metric that bet measures, the new leading total, and the
  game that changed it — not just the scoreline. Use the correct unit per bet:
  Goal Machine = goals scored · The Sieve = goals conceded · Card Sharks = yellow cards ·
  Seeing Red = red cards · Wrong Net = own goals · Spot-Kick Magnet = penalties conceded ·
  Corner Shop = corners won · The Hatchet Men = fouls · The Wall = clean sheets ·
  Aerial Threat = headed goals · Super Sub = sub goals · Tiki-Taka = average possession % ·
  Shooting Gallery = shots faced · The Hammering = scoreline/margin · Off Like a Rocket = goal minute ·
  Fergie Time = goal minute · Comeback Kings = goals overturned · Smash & Grab = winning possession % ·
  All Sizzle No Steak = shots per goal · Total Domination = highest match possession %.
  Good: "Corner Shop: now Brazil (Sam) on 10 corners, their 4-0 v Haiti did it."
  Bad:  "Corner Shop: now Brazil (Sam)."  (no metric, no reason)
- Movers layout — group by punter, biggest mover first: a BOLD headline line per punter (name + new total
  + one-line reason), then each bet they took on its OWN bulleted line "• {Bet}: {flag}{Team} ({metric})".
  Roll tied/shared bets into one "Shared:" line at the end. Keeps it scannable, not a paragraph.
- On today layout — lead with the TEAMS, then the time, then stakes underneath:
  "{flag}{Home} v {flag}{Away} — {kickoff} BST" then a short who-owns-what line below it.
- If nothing changed: "*💥 Movers* — quiet day, no shake-ups on the board."

Template (drop any empty section):

*🏆 WC2026 SWEEPSTAKE — {Friday 12 June}*

*📋 Results since yesterday*
{home flag} {Home} {hs}-{as} {Away} {away flag}
...
{one short banter line}

*💥 Movers*
🥇 *{Punter} £{n}* — {one-line reason}
• {Bet}: {flag}{Team} ({metric})
• {Bet}: {flag}{Team} ({metric})
*{Punter} £{n}* — {reason}
• {Bet}: {flag}{Team} ({metric})
Shared: {Bet} ({flag}/{flag} {metric} each) · {Bet} ({flag}/{flag} {metric} each)

*💰 Money table*
1. *{Punter} £{n}*
2. {Punter} £{n}
3. {Punter} £{n}
=. {everyone still on zero} — all £0
{one-line dig at the bottom}

*⚽ On today*
{flag}{Home} v {flag}{Away} — {kickoff} BST
{who owns each side, what's at stake}
...
👀 *One to watch:* {punter} leaps into 1st if {flag}{Team} {does X}.

📲 Live board: https://leo-gecco.github.io/wc-sweepstake/

# DO NOT
- Do not change the ALLOC draw, the sideBets list, or any rendering/logic. You may only touch:
  `recentScores` (add rows), `fixtures` (remove played games), `lastUpdated`, and `whatsapp/{date}.txt`.
- Do not write more than one `whatsapp/{date}.txt` per UK day (the MESSAGE GATE enforces this).
- Do not add OR partially update a match that is not FULL-TIME (live, half-time, or scheduled). Wait until
  the summary status is final ("post"/completed) — see ONLY-WHEN-FINISHED above.
- Do not duplicate a match already present.
- Do not push a file that fails validation.
```

---

## GitHub auth / pushing
- We're using **git push with the token inline in the prompt** (see PUSH METHOD). The connector is read-only
  on this repo and routine secrets aren't available, so this is the route. The token is the leo-gecco PAT
  with write access — the one proven to push to this repo all along.
- **Rotate this token after the tournament.** It sits in the prompt and run logs, so keep it fine-grained
  (Contents: R/W on this one repo) and revoke/replace it once the sweepstake is over.
- The routine must CONFIRM a fresh commit is on `main` before reporting success (never trust a tool's
  return alone — the first run falsely thought it had pushed).
- If git push 403s with "git proxy token has no push scope", the container is overriding the inline token
  with its own read-only one — then the routine can't publish at all and we move to GitHub Actions below.

## If pushing from the routine is impossible: GitHub Actions (robust, no token juggling)
A scheduled GitHub Actions workflow in this repo can do the whole job — fetch ESPN, update dashboard.html,
write the daily message, and commit — using the Action's built-in `GITHUB_TOKEN`, which always has write
access to its own repo. No PAT, no connector, no proxy. The trade-off: the data update is deterministic
Node (no AI), so the WhatsApp text would be templated rather than witty. A hybrid is possible: Actions
handles the data + commit; the AI routine is kept only to draft the fancy WhatsApp message (printed in its
run output for you to copy). Ask and I'll build the Actions workflow.

## 3. Notes

- **Cadence:** board refreshes every 2 hours through the evening, then a single 08:00 BST run
  (`0 7,17-23/2 * * *`); the WhatsApp message is written once a day by that 08:00 run (MESSAGE GATE). The
  evening cadence keeps the live board fresh while games are being watched; after midnight there are no
  runs, so overnight games (kickoffs as late as 05:00 BST) are caught and recapped by the morning run. The
  gate means just one morning message, no chat spam. Most runs find nothing new and make no commit, so it
  stays cheap.
- **Groups:** the prompt tells Claude to use official FIFA groups / ESPN standings. Once the real draw
  is confirmed, paste the fixed team→group map in to remove any ambiguity.
- **First run:** trigger it manually once to confirm ESPN is reachable from the routine container and the
  push to `main` works, before relying on the schedule.
- **Tonight (11 Jun, first real test):** the opener is Mexico (Tom) v South Africa (Callum), Group A,
  20:00 BST. It should finish ~21:50 BST, so trigger the routine MANUALLY at ~22:30 BST rather than waiting
  for the schedule. Expect: 1 row added (Group A, 11 Jun), lastUpdated stamped with the UK time, the played
  fixture trimmed from `fixtures:`, a commit to `main`, and a WhatsApp draft naming Tom & Callum. If ESPN's
  summary still shows the game unfinished, wait 30 mins and re-run (idempotency means re-runs are safe).
- **Moment bets** (fastest goal, latest winner, comeback) are the trickiest to derive; if they ever look
  off, they can be corrected by hand — everything recomputes from the data on the next page load.

## 4. Avoiding the "can't find the games" problem + how to test

Two things caused trouble in manual testing; both are handled by the routine, not by you:

- **Right date, automatically.** The routine derives yesterday/today (UTC) from its schedule — you never
  type a date. It fetches a 2-day window and dedupes, so late-finishing games and timezone edges are
  caught on the next run.
- **Right data, via the summary endpoint.** The scoreboard finds the fixtures; the summary endpoint gives
  the real completed stats (see DATA SOURCE). This is the fix for "the game looked unplayed."

**No setup change is needed** — same repo, schedule, and permissions. Only the prompt changed (the
two-step scoreboard→summary flow). Re-paste the updated prompt if you already created the routine.

**How to test it (before the World Cup starts):**
1. Temporarily change the league slug in the prompt from `fifa.world` to `fifa.friendly` and point it at a
   day with finished friendlies (e.g. 2026-06-10). Trigger the routine manually.
2. Confirm the run: (a) it listed the day's games, (b) it opened the summary per game and pulled real
   stats, (c) it wrote rows + updated lastUpdated, (d) it committed to `main` and the live site changed,
   (e) it produced a sensible WhatsApp draft with metrics. Friendly teams won't be in your ALLOC draw, so
   they'll show as "manual review" flags — that's expected and proves the name-matching guard works.
3. Run it a SECOND time on the same date — it should add nothing (idempotency working).
4. Revert the slug to `fifa.world`. On 11 June, do one more manual run as the real first-day check.

If steps 2-3 pass on a friendly date, the live World Cup runs will behave the same.

## Alternative (no AI, deterministic)
If you'd rather not spend tokens per run, the same job can be done by a **GitHub Actions** cron workflow
running a small Node script (fetch ESPN → parse → update the file → commit). It can run more frequently
and costs nothing, but it's fixed logic with no judgement for edge cases. Say the word and
I'll build that version instead.
