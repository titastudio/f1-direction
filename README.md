# FORMULA ONE DIRECTION — Project Handoff Document

A single-file HTML/CSS/JS racing career life-sim, pixel/Tamagotchi aesthetic.
Written so a fresh session can start working without re-reading the codebase.

Current deliverable: `formula-one-direction.html` in `/mnt/user-data/outputs/`.

**Keep this document current.** Every fact below was grep-checked against
the code as of Pass 18. If you find something here that's wrong, it's a
documentation bug — fix the doc, don't just work around it silently. Before
ending any session that touched code, update STATE SHAPE, QUICK-REFERENCE,
REMAINING BACKLOG, and add a CHANGELOG entry. A stale handoff doc costs the
next session more time than writing an accurate one costs you now.

---

## BEFORE YOU START — 5-STEP CHECKLIST

1. **Copy source files** from `/home/claude/apex/` (or wherever this session's
   files landed) — never edit `index.html` / the delivered `.html` directly.
   If files are missing, see "Recovering lost source files" below.
2. **Rebuild + baseline test** before touching anything:
   ```bash
   cd /home/claude/apex && python3 rebuild.py
   node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>([\s\S]*?)<\/script>\s*<script>/);fs.writeFileSync('extracted.js',m[1]);"
   node --check extracted.js && echo OK
   for t in regression_test coverage_audit retirement_test suite_economy_test suite_progression_test suite_features_test suite_features2_test suite_newfeatures_test suite_pass21_test; do
     timeout 120 node $t.js 2>&1 | tail -5
   done
   ```
   Confirm everything is clean BEFORE making changes, so any later failure is
   known to be yours, not a pre-existing issue. Current clean-baseline check
   counts: `suite_economy_test.js` 23, `suite_progression_test.js` 9,
   `suite_features_test.js` 20, `suite_features2_test.js` 8,
   `suite_newfeatures_test.js` 14, `suite_pass21_test.js` 4 (plus the 3
   structurally-separate scripts: regression_test, coverage_audit,
   retirement_test). `suite_features_test.js` was split into two files in
   Pass 22 — see TESTING WORKFLOW's gotcha note for why.
   **Re-count with `grep -c "await runCheck" suite_*.js` rather than trusting
   this number** — it drifts the moment someone adds a check without
   updating this doc.
3. **Read "ARCHITECTURE — LOAD-BEARING PATTERNS"** below — every pattern
   listed there caused a real shipped bug before and will bite you again if
   skipped.
4. **Pick your task** from "REMAINING BACKLOG" below, or scope a new one.
5. **After every change**: rebuild → syntax check → full test suite again.
   Add a test for whatever you built (see EXTENSION CHECKLIST). Don't call a
   change done until the suite is clean. **Update this doc** before ending
   the session.

### A note on test flakiness
This sandbox's headless Chrome can disconnect mid-suite under sustained load
(see TESTING WORKFLOW gotchas below). If a run shows 1-3 failures in
`suite_features_test.js` with an error like `Protocol error`, `detached
Frame`, or `Execution context was destroyed`, re-run the suite once or
isolate the failing check in its own `puppeteer.launch()` before assuming
it's a real regression — this has repeatedly turned out to be sandbox
noise, not broken code, across many past sessions. Confirm by checking
whether the SAME check fails consistently across 2-3 runs (likely real) or
different checks fail each time (sandbox noise).

---

## HOW THE PROJECT IS ASSEMBLED

```
shell.html      — HTML skeleton with {{STYLES}} / {{SCRIPTS}} placeholders
styles.css      — all CSS (design tokens, pixel UI components, layout)
data.js         — static game data: TEAMS (+ per-team dev state), tracks,
                  sponsors, name pools, radio lines, traits/archetypes,
                  romance/celebrity pools, quiz questions, rivalry origins,
                  sponsor goal-completion dialogue
engine.js       — pure logic: driver generation, race/qualifying math, AI
                  grid driver skill drift, save/load, constants,
                  retirement/turnover, protégé system
app_core.js     — App singleton, boot screen, character creation,
                  editable-name system, TEAMS<->state sync, skill-cap
                  enforcement, team morale
app_actions.js  — modal system + Training/Lifestyle/Relationships/Media/
                  Sponsors/Mentor modals, first-time explainers
app_hub.js      — Home screen (the Tamagotchi "pod" hub) + day/week loop
app_race.js     — Race Center screen + practice/qualifying/race sim +
                  results + trait evolution + reputation archetypes +
                  rivalry system
app_screens.js  — Team&Car, Career Stats, Journal, Museum, Save Menu, Season
                  End, Contract Offers, Retirement, Season Objectives
app_tutorial.js — onboarding tutorial + Help modal reference content
rebuild.py      — concatenates JS files + injects into shell.html → index.html
```

**Rebuild**: `cd /home/claude/apex && python3 rebuild.py`. Reads `shell.html`,
replaces `{{STYLES}}`/`{{SCRIPTS}}`, writes `index.html`.

### Recovering lost source files
If source files are missing (container reset) but a delivered
`formula-one-direction.html` exists, extract all JS as one blob and re-split
manually by searching for `FORMULA ONE DIRECTION — <SECTION NAME>` comment
headers that mark each file's start:
```js
node -e "
const fs = require('fs');
const html = fs.readFileSync('formula-one-direction.html','utf8');
const m = html.match(/<script>([\s\S]*?)<\/script>\s*<script>/);
fs.writeFileSync('extracted-all-js.js', m[1]);
"
```

---

## TESTING WORKFLOW

Puppeteer + bundled Chrome are available:
```
Chrome:     /home/claude/.cache/puppeteer/chrome/linux-131.0.6778.204/chrome-linux64/chrome
Puppeteer:  /home/claude/.npm-global/lib/node_modules/@mermaid-js/mermaid-cli/node_modules/puppeteer
```
```js
const puppeteer = require('/home/claude/.npm-global/lib/node_modules/@mermaid-js/mermaid-cli/node_modules/puppeteer');
const CHROME = '/home/claude/.cache/puppeteer/chrome/linux-131.0.6778.204/chrome-linux64/chrome';
const browser = await puppeteer.launch({headless:'new', args:['--no-sandbox','--disable-dev-shm-usage','--disable-gpu','--disable-extensions'], executablePath:CHROME});
```
No `page.waitForTimeout()` in this Puppeteer version — use
`await new Promise(r=>setTimeout(r, ms))`.

**Gotcha — always scroll before clicking.** Use a `robustClick()` helper that
calls `el.scrollIntoView({block:'center'})` before every click, or clicks under
the fixed `#bottom-nav` silently miss and look like a stuck game loop.

**Gotcha — headless Chrome disconnects under sustained load.** This sandbox's
Chrome can disconnect at any point in a single browser session (varies run
to run — a Puppeteer/Chrome sandbox limitation, not a game bug). Confirmed
repeatedly across sessions: the SAME full-suite run can show different
checks failing each time with `Protocol error`/`detached Frame`/`Session
closed`/`Execution context was destroyed`, and isolating a "failing" check
into its own single-page script almost always shows it passing cleanly.
**Pass 22 confirmed the root cause at the hardware level: this sandbox has
exactly 1 CPU core (`nproc` → 1).** Headless Chrome is itself multi-process;
a single core under sustained load (many `puppeteer.launch()` calls across
a long session, or a heavy `page.evaluate()` computation loop right before
a UI-interaction check) will predictably starve and disconnect. This is not
fixable from test code alone — it's a property of the sandbox, not the
suite. Mitigations already in the test suite:
- Wrap long loops in try/catch, treat those four error strings as a sandbox
  crash (report separately, don't fail the whole test on it).
- Keep any single playthrough's target modest (regression_test.js targets 5
  rounds, ~85 iterations) — split into multiple `puppeteer.launch()` calls or
  fast-forward state directly via `page.evaluate()` (see retirement_test.js,
  which jumps `calendar.currentRound` to the season's last round instead of
  clicking through ~200 iterations) rather than raising the budget and hoping.
- Launch with `args:['--no-sandbox','--disable-dev-shm-usage','--disable-gpu','--disable-extensions']`.
- **(Pass 22)** If a single `suite_*.js` file consistently disconnects from
  the SAME point onward across 3+ runs (not different points each time —
  that's noise; the SAME point is a genuine overload trigger), split the
  file into two `puppeteer.launch()` calls at that boundary rather than
  chasing timing margins. `suite_features_test.js` (28 checks) was split
  into `suite_features_test.js` (20) + `suite_features2_test.js` (8) this
  way — the second half consistently disconnected right after a
  1500-trial `page.evaluate()` computation, and splitting fixed it outright
  where a stabilization delay alone did not.
- **(Pass 22)** A cached Puppeteer `ElementHandle` (from `page.$()`) can
  silently no-op on `.click()` if grabbed even slightly before use, even
  though the element is genuinely present/clickable a moment later via a
  FRESH lookup — confirmed via direct A/B testing (same click, same state,
  only the handle's age differs). Prefer
  `await page.evaluate(() => document.getElementById(id).click())` over a
  cached handle for any click after a retry-poll or a preceding heavy
  computation.
- **(Pass 22)** Before a long testing session, clean up
  `/tmp/.org.chromium.*` directories periodically (`rm -rf
  /tmp/.org.chromium* 2>/dev/null`) — dozens of leftover profile dirs from
  earlier `launchBrowser()` calls accumulate over a long session and can
  compound the single-core contention above.

**Gotcha — statistical checks need enough trials, and must exclude
sentinel/outlier values from averages.** A DNF's `raceScore` is a `-9999`
sentinel; even a ~5% DNF rate will swing (and can flip the sign of) a
few-hundred-trial average of a small true signal. When averaging a
probabilistic effect, filter out DNFs/other sentinels first unless DNF rate
itself is what you're measuring, and use enough trials (1000+) that a small
true effect isn't drowned by noise — several existing checks (e.g. the Tyre
Whisperer DNF-rate check in `suite_economy_test.js`) have shown intermittent
false failures at 4000 trials on a true effect under 1 percentage point.

**Gotcha — a first-time-explainer modal firing mid-test-loop can silently
break unrelated checks.** `showFirstTimeExplainer()` (`app_actions.js`)
replaces the current modal in place. If it's wired with any `setTimeout`
delay after a `closeModal()`, it can land a beat after a test's click-loop
has already moved on, breaking the NEXT iteration's element lookup instead of
the one that triggered it. Fire these synchronously, in place of the plain
`closeModal()`, only the first time.

### Test suite (9 runnable scripts + 1 shared helper module)

```
_test_helpers.js         NOT a test -- shared launch/robustClick/startNewCareer/
                          runCheck/printSummaryAndExit helpers. Required by every
                          suite_*.js/regression_test.js/retirement_test.js/
                          coverage_audit.js file. As of Pass 22, startNewCareer()
                          also skips the junior series prologue (feature idea
                          #12) and waits out maybeShowTutorial()'s own 400ms
                          setTimeout before checking for the tutorial skip
                          button -- a 300ms wait there was an intermittent
                          race that left the tutorial modal open and silently
                          blocking every subsequent click in a test.

regression_test.js       full playthrough: new career, sponsor sign, race loop, zero console errors
coverage_audit.js        4 viewports (360×640, 375×667, 390×844, 1440×900) × every screen/modal,
                          checks no overflow under #bottom-nav, no horizontal scroll, close button reachable
retirement_test.js       force-age retirement flow → legacy perk → RETIRES screen → Legend Exhibition (Pass 22)

suite_economy_test.js    23 checks -- car development, facility effects, training/setup-confidence
                          pacing, performance-clause contracts, sponsor rework (Pass 22: 10-goal
                          progression now QUEUES a renewal choice instead of auto-converting to
                          permanent -- see feature idea #3), difficulty knobs, stat/archetype wiring
                          (mood, mediaReputation sponsor unlock, Tyre Whisperer DNF relief)

suite_progression_test.js 9 checks -- celebrity encounters, romance overhaul, multi-career Hall
                          of Fame, legacy-perk legend-driver carryover, difficulty-gated starting
                          teams, mentor/protégé graduation (Pass 22: now reads p.protegeRoster
                          rather than assuming p.protege still points at the just-graduated
                          prospect -- see feature idea #16's roster auto-advance), trait-evolution
                          fixes (Ice Cold/Late Braker/Workhorse/Hot Head, plus a false-positive guard)

suite_features_test.js   20 checks -- race-weekend depth features, the must-resolve-modal fixes,
                          22-driver grid invariant, sprint weekend flow, "current pace modifiers"
                          chip. Split from a single 28-check file in Pass 22 (see TESTING WORKFLOW's
                          gotcha note) -- checks 1-20 here, 21-28 in suite_features2_test.js.

suite_features2_test.js  8 checks (Pass 22 split) -- safety car banner, formation-lap mini-moment,
                          plausibleLapTime UI rendering, stat-attributed radio lines, weather-shift
                          events, Q1/Q2/Q3 elimination framing. Kept in its own browser session
                          specifically because this half consistently disconnected the shared
                          session when combined with the first half's heavy trial loops.

suite_newfeatures_test.js 14 checks (Pass 22) -- targeted checks for the highest-risk/most
                          load-bearing of the 17 new features: junior series prologue (state-
                          creation-order dependency, bounded stat adjustment), Team Ownership Mode
                          (silent auto-resolve chain, season-wrap re-surfacing via a REAL
                          coordinate-click test, modal-overlay correctly blocking a stale click),
                          driver academy roster + grid invariant, contract release clause, sponsor
                          renewal negotiation, multi-year roadmap milestones (met + missed paths),
                          team orders detection/swap, weather microclimates fire-rate, sim racing
                          side career + cross-over sponsor unlock, track records board.

suite_pass21_test.js     4 checks (Pass 22) -- closes the Pass 21 "Not done" gap: quiz double-click
                          re-entry (rapid double-click on the last question doesn't throw/corrupt
                          state), protégé name uniqueness (400 trials, zero collisions), Constructors'
                          Championship points staying with the team that earned them after a
                          mid-season switch, rival-is-teammate dedupe in the multi-driver press
                          conference (no doubled effects, no duplicated name). Kept in its own
                          browser session (not folded into suite_features_test.js) per the same
                          sustained-load reasoning as the suite_features2_test.js split.
```

Run each with a `timeout` wrapper — 9 commands total instead of running every
check file individually:
```bash
for t in regression_test coverage_audit retirement_test suite_economy_test suite_progression_test suite_features_test suite_features2_test suite_newfeatures_test suite_pass21_test; do
  timeout 120 node $t.js 2>&1 | tail -5
done
```
A `suite_*.js` failure prints exactly which named check(s) failed
(`FAIL: suite_X -- N check(s) failed: <names>`) so you don't need to dig
through full output to find the broken one.

**Adding a new check**: put it in whichever suite it thematically fits (or
start a new `suite_*.js` if none fit), as one more `await runCheck(results,
'name', async () => { ... return {pass, detail}; })` block — copy an
existing block's shape (`newPage`/`startNewCareer`/assertions/`page.close()`/
push to `allErrors`) rather than inventing a new pattern. **Update the check
counts in this section when you do.** If the suite you're adding to already
has 20+ checks or any heavy (1000+ trial) `page.evaluate()` loops, consider
starting a new file instead of growing it further — see the sustained-load
gotcha above.

---

## DESIGN LANGUAGE (don't drift without deliberate reason)

**Aesthetic**: pixel/handheld-console + Tamagotchi digital pet, NOT a
realistic racing-sim look.

**"The pod"**: circular screen-in-screen (`.pod`/`.pod-wrap`) showing the
driver as a big emoji sprite with idle bounce + mood badge. 150px on Home,
96px on Race Center/Team.

**Device shell**: `#device` is a fixed-width (max 430px) card; centers with
rounded corners on wider viewports rather than stretching.

**Colors** (`:root` in `styles.css`): `--shell` mint green, `--screen-bg`
cream, `--ink` warm dark brown, `--candy-pink/yellow/mint/blue/purple/red`
accents (`.gold` = primary CTA). Fonts: `'Press Start 2P'` (pixel display) +
`'Nunito'` (body).

**Reusable components** — always use these, don't reinvent:
`.pixel-panel`, `.pbtn`/`.pbtn.gold/mint/pink/blue/purple/small/full`,
`.chip`/`.chip.gold/pink/mint/blue/red`, `.pixel-bar-track`+`.pixel-bar-seg`
(via `App.renderPixelBar(value, colorClass)`), `.modal-overlay`/`.modal`/
`.modal-header`/`.modal-body`/`.modal-close`, `.option-grid`/`.option-card`,
`.editable-name`, `.headline-card`, `.toast`/`.toast-wrap`.

**Bottom nav**: 6 fixed tabs (Home/Race/Train/Team/Social/Stats), wired once
globally at `App.init()` via `wireGlobalNav()`.

**Guide/help text formatting**: flowing paragraphs, not `<br>`-separated
line fragments. Keep it short and scannable, but as complete sentences —
this was tried both ways (Pass 14 → line breaks, Pass 16 → reverted to
paragraphs) and paragraphs read as cleaner for this UI.

---

## ARCHITECTURE — LOAD-BEARING PATTERNS

### The App singleton
Everything hangs off global `App` (`app_core.js`). Every file does
`Object.assign(App, {...methods...})`. Method names must be unique across
files — a collision silently overwrites with no error. `App.state` holds the
whole save-game (with one deliberate exception — see "TEAMS is NOT part of
App.state" below).

### TEAMS is NOT part of App.state — it's synced, not stored
`TEAMS` (`data.js`) is a plain module-level constant, mutated at runtime
(`carStrength`/`reliability`/`upgrades`/`devSlotsUsed` all change from AI
development, player car upgrades, and regulation changes). It is **not**
saved directly — `App.saveState()` calls `syncTeamsToState()` first, which
snapshots the mutable fields into `state.teamState` right before every save;
`App.init()`'s load path calls `syncTeamsFromState()` to restore `TEAMS` from
that snapshot right after load. If you add a new mutable field to `TEAMS`
entries, it MUST be added to both sync functions (`app_core.js`) or it will
silently reset to its hardcoded starting value on every page reload.

### saveState() is the central choke point — use it for cross-cutting rules
`App.saveState()` (`app_core.js`) runs `enforcePlayerSkillCap()`, then
`syncTeamsToState()`, then `saveGame()`, on every single call. Any rule that
needs to apply after EVERY state mutation regardless of which of the 20+
call sites triggered it belongs here, not threaded through each site
individually — that's how the skill cap (see STATE SHAPE below) enforces
itself without needing to touch every `p.attributes.X = clamp(...)` call in
the codebase. If you add another cross-cutting invariant, follow this same
pattern rather than scattering the check.

### Bottom nav — wired ONCE, globally
**Past bug**: per-screen nav wiring meant two screens forgot to re-wire it,
so navigating there dead-ended. **Fixed**: `App.wireGlobalNav()` runs once in
`App.init()`, targets the persistent `#bottom-nav [data-nav]` elements (never
destroyed). A new full screen only needs to toggle `.nav-item.active` — never
add nav click wiring again.

### Modal system — sticky header, scrolling body
**Past bug**: the whole `.modal` scrolled as one block, so long content
scrolled the ✕ close button out of view. **Fixed**: `App.openModal(html)`
finds `.modal-header` in the given HTML and moves everything after it into an
auto-injected `.modal-body` (CSS: header `flex-shrink:0`, body
`overflow-y:auto`). Every modal's content string must start with
`this.modalHeader(title)` — never build a modal a different way.

### Must-resolve modals — don't hide the X reflexively
**Past bug (critical, confirmed reproducible)**: several result modals
(race/qualifying/sprint results, Contract Offers when the contract has
already expired) let a player X-close them via the generic
`wireModalClose()` pattern, even though state had ALREADY been advanced
before the modal opened (round marked complete, calendar index moved
forward). Closing without resolving desynced the hub from the calendar and
could leave the game racing the wrong track under a stale banner, or let a
player continue indefinitely with no valid contract. **Fixed**: these modals
hide the close button and disarm the overlay click, exactly like the
podium-interview/quiz/season-wrap modals already did — they must be resolved
through their own button(s). When adding a new modal, ask: "has state already
moved forward before this modal opened, in a way the player can't undo by
just closing it?" If yes, make it must-resolve.

### First-time explainer pattern
`App.showFirstTimeExplainer(flagName, icon, title, text, onContinue)`
(`app_actions.js`) shows a small dismissable info modal the FIRST time a
player hits a mechanic that would otherwise only be documented in the Help
modal, gated by a `p[flagName]` boolean (same shape as `p.tutorialSeen`).
Call it BEFORE the real modal/content would have opened — it renders in
place, and `onContinue()` fires on dismiss to proceed to whatever would have
shown anyway; if already seen, `onContinue()` fires immediately with no
modal. Six currently wired: `seenStatEffectExplainer` (first training
result), `seenMentorExplainer` (first protégé recruitment),
`seenRomanceExplainer` (first nightlife candidates), `seenSponsorExplainer`
(first sponsor signing), `seenDevSlotsExplainer` (first Team & Car visit),
`seenCelebrityExplainer` (first celebrity encounter). Grep
`showFirstTimeExplainer(` call sites for the current full list rather than
trusting this line if it's been a while. **Fire it synchronously, never
behind a `setTimeout` delay** — see the TESTING WORKFLOW gotcha above.

### Universal editable-name system
`this.editableName(id, kind, currentName)` (`kind`: `'player'|'driver'|'npc'`)
renders a rename-on-click span. After injecting HTML containing one, call
`this.wireEditableNames(container)` (defaults to `document`) or the pencil
icon is inert. `openModal()` already does this automatically.

### Save/load
`localStorage` key `'formulaonedirection_save_v1'` (renamed from the
pre-rename `'apexpal_save_v1'`). `loadGame()` (`engine.js`) falls back to the
old key once and migrates it forward if the new key is empty, so no existing
save is lost — new saves are only ever written under the new key.
`saveGame()`/`loadGame()`/`deleteSave()` are plain functions in `engine.js`.
Always call `this.saveState()` after mutating `App.state` (wraps the skill
cap, `syncTeamsToState()`, and `saveGame()` — see "saveState() is the
central choke point" above) — never call `saveGame()` directly.
`LEGACY_KEY`/`HOF_KEY` got the same rename+migration treatment in Pass 19
(see CHANGELOG) — `'formulaonedirection_legacy_v1'`/`'formulaonedirection_hof_v1'`,
each falling back to their own pre-rename `apexpal_*` key once.

### Rebuild-safe editing
Always `view` the exact current file section immediately before editing with
`str_replace`-style tools. Stale/remembered content has previously deleted a
needed function, left orphaned duplicate code, or (once) swallowed another
object literal's closing brace mid-edit — all three broke the game and were
only caught by the syntax check or regression test, not by eye. **Syntax
check is necessary but not sufficient — always also run the regression
test.**

---

## STATE SHAPE (`App.state`)

Re-derived directly from `createPlayer()`/`createDriverAttributes()`
(`engine.js`) and the `this.state = {...}` block in `App.startNewCareer()`
(`app_core.js`) — every field below is confirmed live in current code.

```js
{
  player: {
    id:'player', name, gender, nationality, flag, age, teamId,
    attributes: {
      driving: { racePace, qualifyingPace, cornering, braking, starts,
                 overtaking, defending, wetSkill, tyreManagement,
                 fuelSaving, consistency, reaction },   // 0-99, SEE SKILL CAP below
      mental:  { confidence, focus, pressure, leadership, discipline, adaptability }, // 0-99, SEE SKILL CAP below
      lifestyle: { fitness, mood, stress, energy },      // 0-100, NOT subject to the skill cap
    },
    fanPopularity, mediaReputation, traits:[], reputationArchetypes:[],
    drivingStyle: null|'Aggressive'|'Defensive'|'Tyre Saver', styleProgress:{},
      // only these 3 values are ever player-earned (recordStyleProgress(),
      // engine.js, via 5+ uses of the matching race approach); the rest of
      // DRIVING_STYLES (data.js) is AI-grid-only, rolled at driver creation.
      // Has a real (small) mechanical effect in simulateRace() --
      // NOT purely cosmetic. radioLine() (data.js) maps a driving style
      // through DRIVING_STYLE_TO_PERSONALITY before bank lookup.
    contract:{teamId,years,salary}, money, homeLevel:1-6,
    careerStats:{starts,wins,podiums,poles,fastestLaps,dnfs,points,wetWins},
      // fastestLaps: incremented by rollFastestLap() (engine.js), called
      // from executeRace() (app_race.js) right after simulateRace(); feeds
      // a 5-count milestone. wetWins feeds a 3-count milestone. Both shown
      // in Career Stats (app_screens.js renderCareerStatsModal()).
    trackFamiliarity:{[trackId]:65-100},
    mechanicalRiskMult: number|undefined,
      // multi-weekend DNF consequence chain (Pass 19) -- transient field,
      // same lifecycle shape as familiarityBonus (set before simulateRace,
      // read inside it). 1.6 right after a mechanical DNF, decays 0.25/
      // weekend, clearable to 1 via Team screen's "Address Reliability
      // Concern" button. undefined/1 = no elevated risk.
    isHomeTrackBonus: number|undefined,
      // Home Tracks (Pass 19) -- transient field, set alongside
      // familiarityBonus at qualifying/sprint-qualifying time when
      // isHomeTrack() (engine.js) is true for that track.
    sponsors:[], sponsorHistory:[{name,season}], unlockedSponsorTiers:[1],
    milestones:[], journal:[{season,week,title,text,icon}],
    seasonAwardsWon:[{season,award}], retired:false,
    difficulty:'Casual'|'Standard'|'Realistic',
    setupTrainingByTrack:{[trackId]:number},  // per-track sim training count, feeds setupHintConfidence -- NOT global
    trainingActionsThisWeek:number, trainingWeekTracker:number|undefined,  // weekly training cap (3/week), tracker resets against s.week
    recentQuizIds:[ids],
    protege: null|{name,nationality,flag,mentorProgress:0-100,graduated:boolean,_academyId},
      // recruit at homeLevel>=3; the SINGLE actively-mentored prospect --
      // see protegeRoster below for the Pass 22 academy roster this now
      // sits alongside. Graduates ONLY into a real grid retirement slot via
      // processGridRetirements(), never as an extra 23rd driver.
    protegeRoster: [ /* up to ACADEMY_MAX_SIZE=3 protege-shaped objects */ ],
      // Driver Academy (Pass 22, feature idea #16) -- deepens the mentor
      // system rather than replacing it. p.protege is always one member of
      // this array (or null). Non-active members still accrue slow
      // "latent" mentorProgress via mentorProtege()'s own session (see
      // app_actions.js). processGridRetirements() (engine.js) scans this
      // WHOLE array for a graduated prospect, not just p.protege, since
      // more than one can reach 100 independently over time.
    joinedTeamOnSeason: number,
      // season the player joined their CURRENT team; a real team switch
      // resets this and costs morale (team-switch cooldown), a same-team
      // contract renewal does not
    releaseClauseUsedThisContract: boolean,
      // Contract release clause (Pass 22, feature idea #9) -- true once the
      // player has requested an early release from the CURRENT contract;
      // resets to false on every new contract signed (renewal or switch).
      // See renderReleaseClauseConfirmModal() (app_screens.js).
    simRacingRep: number,
      // Sim racing side career (Pass 22, feature idea #15) -- uncapped,
      // purely additive reputation grown via Free Time's Sim Racing
      // activity. Gates the Circuitworks cross-over sponsor (SPONSOR_POOL,
      // data.js, needsSimRacingRep:true) at >=40 via sponsorUnlocked().
    isTeamOwner: boolean,
      // Team Ownership Mode (Pass 22, feature idea #13) -- set once at
      // character creation, never changes mid-career. When true, race
      // weekends auto-resolve (see autoResolveOwnerRaceWeekend(),
      // app_race.js) instead of the player choosing strategy.
    contract:{teamId,years,salary,clause,roadmapMilestone:null|{type,targetYear,target,reward,met:null|boolean,signedSeason,podiumsAtSigning}},
      // roadmapMilestone (Pass 22, feature idea #11) -- an OPTIONAL, visible
      // upfront milestone on multi-year (2+) contract offers, distinct from
      // the existing hidden performance `clause`. Checked in
      // handleSeasonEnd() at the stated targetYear; met=true pays out
      // reward.salaryRaise, met=false has NO penalty (pure upside, unlike a
      // missed clause which forces years to 0).
    seenStatEffectExplainer, seenMentorExplainer, seenRomanceExplainer,
    seenSponsorExplainer, seenDevSlotsExplainer, seenCelebrityExplainer: boolean|undefined,
      // one flag per first-time-explainer -- see ARCHITECTURE above for the
      // full current list; grep `showFirstTimeExplainer(` to double check
  },
  trackRecords: {[trackId]:{driverName,lapTimeText,season,isPlayer:boolean}},
    // Track records board (Pass 22, feature idea #4) -- fastest lap EVER
    // recorded at each circuit this save, any driver, updated in
    // finishExecuteRace() (app_race.js) right after rollFastestLap(), only
    // overwritten when plausibleLapTime()'s parsed seconds are genuinely
    // lower than the existing record. Museum-only display, never read by
    // any mechanical system.
  pendingSponsorRenewals: [sponsorName, ...],
    // Sponsor renewal negotiation (Pass 22, feature idea #3) -- queued by
    // checkSponsorObjectives() (app_actions.js) the instant a sponsor's
    // 10th goal completes, INSTEAD of auto-converting to permanent.
    // Resolved via renderSponsorRenewalModal() (Lock In = guaranteed
    // current rate, Renegotiate = 55% chance of a real raise / 45% chance
    // goalsCompleted knocks back to 7). Chained into the race-result
    // advance flow right after the podium interview, must-resolve.
  grid: [ /* ~21 AI drivers + player = 22 total, see makeRivalDriver() in engine.js */
    { id,name,teamId,age,nationality,flag,drivingStyle,gender:'M',
      attributes:{...}, fanPopularity, mediaReputation, traits:[],
      reputationArchetypes:[],
      careerStats:{...}, memories:[], rivalryLevel:0-100, respect:50,
        // respect is NOT write-only -- see "Skill cap and dynamic rivals" below
      headToHead: {wins,losses}|undefined,
        // set/updated by updateRivalry() (app_race.js) for any driver who's
        // ever finished adjacent to the player; shown in Rival Profile
      rivalryOrigin: string|undefined,
        // rolled once by rollRivalryOrigin() (data.js RIVALRY_ORIGINS) the
        // moment this driver is FIRST crowned rivalId -- stable for the
        // life of the rivalry, never re-rolled
      teammateTally: {wins,losses}|undefined,
        // Teammate relationship arc (Pass 19) -- only ever set/read on
        // whichever grid driver currently shares p.teamId; same shape as
        // headToHead but THIS-season-scoped (captured + reset to
        // {wins:0,losses:0} in handleSeasonEnd(), app_screens.js, right
        // before processGridRetirements() could wipe it via a driver
        // replacement)
      careerLegacy: {worldTitles,dns,highestSeasonPoints,highestSeasonYear,rookie}|null,
        // Pass 20 -- display-only pre-loaded career extras for NAMED
        // drivers only (TEAM_DRIVER_PROFILES, data.js); null for any
        // driver generated without a matching profile (fresh rookies,
        // retirement replacements). Never read by any mechanical system --
        // see driver.careerStats below for the fields that actually feed
        // game logic.
      isLegend: true|undefined, isProtegeGraduate: true|undefined,
      romance:{status,spark} }
  ],
  season:number,                 // opaque internal counter, starts 2027 — NEVER shown to player
  careerStartSeason:number,      // use seasonLabel()/seasonNumToYear() for display, never raw `season`
  calendar:{ rounds:[{round,trackId,completed,isSprint,weather,trackTemp}], currentRound:index,
             midSeasonDevRoundIndex:index, midSeasonDevApplied:boolean },
    // midSeasonDev* guard the one-time mid-season AI development bump so it
    // can only ever fire once per season
  dayIndex:0-6, week:number, standings:{[driverId]:points},
  constructorPoints:{[teamId]:points},
    // Constructors' Championship (Pass 19/21) -- tracked SEPARATELY from
    // standings, incremented at the exact moment points are scored against
    // whichever teamId a driver was on AT THAT MOMENT (executeRace()/
    // resolveSprintRace(), app_race.js) -- NEVER derive this from summing
    // standings for drivers who currently have a given teamId, that was a
    // real bug (Pass 21) that let a mid-season switch silently move a
    // driver's whole season total to their new team. Reset alongside
    // standings at season start/end.
  newsFeed:['string w/ emoji'], seasonStats:{wins,podiums,points,beatTeammate,q3count},
  seasonObjectives:[{type,text,target,reward:{money,rep}}],   // regenerated each season AND on any team switch
  npcRoster: [ {id,role,name,gender,tenureSeasons,relationship:{friendship,trust,affection,respect,compatibility,memories},
                personality,romance:{status,spark}} ],   // exactly 7 fixed roles
    // personality (Calm/Energetic/Strict/Funny) flavors getTalkLinesForRole()
    // result text for ALL 7 roles (previously Race-Engineer-only)
  history:[{season,team,standing,points}], retiredFlag:boolean,
  rivalId:null|driverId,
  carUpgrades:{aero,engine,chassis,suspension,reliability},
    // LEGACY/UNUSED -- car development now lives on TEAMS[].upgrades
    // (per-team, not per-player). Kept only so old save files still parse;
    // nothing reads this field anymore. Don't add new code that writes to it.
  teamState: {[teamId]:{carStrength,reliability,upgrades,devSlotsUsed}},
    // snapshot of TEAMS' mutable fields, written by syncTeamsToState() right
    // before every save and restored by syncTeamsFromState() right after
    // every load -- see ARCHITECTURE above. This is the ACTUAL persisted
    // source of team development, not carUpgrades above.
  facilities:{gym,simulator,engineering,factory,media,hospitality},  // each 1-5
  currentSetup:{downforce,balance,gearRatio}, tutorialSeen:false,
  teamMorale:{[teamId]:0-100}, lastQualyResults, lastWeather, prevTeamStrength,
    // morale swing MAGNITUDE (not direction) scales with that team's
    // TEAMS[].devTrend.volatility -- see adjustTeamMorale()
    // (app_core.js). A "high pressure"/volatile-culture team's morale
    // swings harder on the same event than a "patient, loyal" team's does.
  pendingTierOffer:true|undefined,
  cosmetics:{careerStartSeason,equipped:{helmet,livery,suit,gloves},
             totalRoundsCompletedThisSeason, lastKnownUnlocked:[ids]},
  careerRetirementCount:number, metPartner:null|{...},
  pendingRomanceCandidates:null|[{...}],  // set when a nightlife outing surfaces candidates, cleared once one is picked
  scoutingOffer:null|{teamId,years,salary,reason},  // unsolicited mid-contract offer from checkScoutingInterest(), engine.js
  seasonStartCareerWins:{[driverId]:number},  // snapshot at season start, used to compute "beat teammate this season" style deltas
  lastRaceHeadline:null|{track,position,dnf},  // feeds the Home hub's most-recent-result display; cleared at season start
  abandonedDriverNames: string[],
    // Pass 20 -- names permanently removed from this career's named-driver
    // pool by a teammate pick at character creation (renderTeammatePickModal(),
    // app_core.js -- generateGrid()'s abandonedName). Checked by
    // generateFreshDriverName() (engine.js) so an abandoned choice can never
    // resurface later as a "fresh" rookie replacement either. Set once at
    // career start, never mutated after (a mid-career switch's un-picked
    // driver is NOT added here -- see rebalanceGridForTeamChange()'s
    // keepDriverId, which handles that case differently, by replacement not
    // erasure).
}
```

**TEAMS** (`data.js`, module-level constant, NOT inside `App.state` — see
ARCHITECTURE above): `{id,name,flag,color,culture,carStrength,reliability,
tier:1-3,devTrend:{ambition,volatility,reliabilityFocus},upgrades:{aero,
engine,chassis,suspension,reliability},devSlotsUsed}`. 11 teams, tiers 1-3.
`culture` is flavor text but `devTrend.volatility` (the number it's actually
describing) has two real mechanical uses: `developTeamsForNewSeason()`'s own
car-strength swings, and team-morale swing magnitude via
`adjustTeamMorale()`.

**Sponsor object** (in `player.sponsors[]`): `{name,type,trait,pay,
needsClean/needsAggression/needsPopularity, objectiveText, objectiveDone,
signedSeason, _attackCount/_streak, _loyaltyMult, goalsCompleted, permanent}`.
10 sponsors defined in `SPONSOR_POOL` (`data.js`), tiers 1-4. Each sponsor
`type` has a matching flavor line in `SPONSOR_GOAL_LINES` (`data.js`),
surfaced via `sponsorGoalLine()` in the player's journal on every goal
completion.

**Legacy record** (`localStorage['apexpal_legacy_v1']`, NOT in `App.state`,
survives `deleteSave()`): `{perkId, retiredPlayerSnapshot}`. Still uses the
pre-rename key name deliberately — only `SAVE_KEY` was renamed (Pass 15);
renaming this one too would need its own migration plan and wasn't in scope.

**Hall of Fame** (`localStorage['apexpal_hof_v1']`, NOT in `App.state`,
survives `deleteSave()` AND new careers — multi-career, append-only except
for the explicit delete-with-confirm flow): array of entries built by
`buildHallOfFameEntry()` (`engine.js`). Same pre-rename key note as above.

### Skill cap and dynamic rivals (Pass 18 — anti-overpower system)
Three linked mechanics, all added together, designed so a player can't grind
every stat to 99 while the rest of the grid stays fixed forever:

1. **Leader-relative skill cap.** `App.enforcePlayerSkillCap()`
   (`app_core.js`) runs on every single `saveState()` call (see "saveState()
   is the central choke point" in ARCHITECTURE above). Every player
   driving/mental stat is capped at the CURRENT points-standings leader's
   same stat + 8 (`SKILL_CAP_MARGIN`), re-evaluated fresh each call — not a
   fixed reference driver. If the player is themselves the points leader,
   the cap uses the best AI driver's stat instead (there's no separate
   fallback needed; the grid's own #1 AI driver already serves that role).
   Per-stat, not "overall rating" — a leader who's elite at braking but weak
   at wet skill produces a correspondingly shaped cap for the player too.
   Lifestyle stats are NOT capped. `App.isNearSkillCap(group, stat)` is a
   read-only helper the Training modal (`app_actions.js`) uses to flag a
   session that's about to hit the ceiling before the player spends
   money/energy on it; `doTraining()` also checks the REAL post-cap value
   (after `saveState()` runs) before writing its result toast, so the
   number shown always matches what actually happened.
2. **AI grid driver skill drift.** `developGridDriversForNewSeason(state)`
   (`engine.js`), called from `handleSeasonEnd()` (`app_screens.js`) right
   after the age++ loop and before `processGridRetirements()`, gives every
   surviving non-legend AI driver a small age-biased season-over-season
   drift on every driving/mental stat: drivers under 24 trend up, drivers
   over 34 trend down, drivers in the 26-30 "peak" window get a small
   roughly-neutral wobble. This means the points leader (and therefore the
   skill-cap reference) genuinely shifts identity over a career instead of
   whoever rolled well at grid generation staying on top indefinitely.
   Legend drivers (`isLegend`) are exempt — they represent an already-earned
   fixed skill level, not an ongoing career.
3. **Player season-end decay.** Also in `handleSeasonEnd()`: every player
   driving/mental stat fades by 2 points at the start of each new season
   (`SEASON_SKILL_DECAY`), representing rust from a season not spent
   training. Smaller than a typical single training session's gain (~2
   before facility bonuses), so regular training more than offsets it — a
   season with no training is where this actually costs something.
   Disclosed to the player via a line in the Season Wrap modal
   (`renderSeasonWrapModal()`, `app_screens.js`).

Player-facing explanation lives in the "Skill Ceiling" Help topic
(`app_tutorial.js`).

---

## QUICK-REFERENCE: FILE → WHAT LIVES THERE

- Team/track/sponsor/name-pool/trait/archetype/rivalry-origin/sponsor-dialogue data → `data.js`
- Race-result math / points / difficulty balance / pace-modifier display → `engine.js` (`driverStrength`, `simulateRace`, `simulateQualifying`, `paceModifierBreakdown`)
- Skill cap enforcement + near-cap UI hint → `app_core.js` (`enforcePlayerSkillCap`, `isNearSkillCap`), called from `saveState()` and `app_actions.js renderTrainingModal()`/`doTraining()`
- AI grid driver skill drift (season-over-season) → `engine.js developGridDriversForNewSeason()`, called from `app_screens.js handleSeasonEnd()`
- Character creation fields → `app_core.js renderCreateCharacter()` + `createPlayer()` in `engine.js`
- New day-action → `app_hub.js renderTodayActions()` + `app_actions.js openActionModal()` + a new `render*Modal()`
- Race weekend flow (practice/qualifying/race/strategy/current-modifiers chip) → `app_race.js renderRaceStrategyPrompt()` and neighbors
- Fastest lap → `engine.js rollFastestLap()`, called from `app_race.js executeRace()`; sprint races deliberately don't roll one (sprints skip all career-stat credit by design)
- Rivalry system (level, origin flavor text, respect's effect on gain rate) → `app_race.js updateRivalry()`/`checkTraitEvolution()`, `data.js RIVALRY_ORIGINS`/`rollRivalryOrigin()`, shown in `app_actions.js renderRivalProfileModal()`
- Sponsor type-specific dialogue → `data.js SPONSOR_GOAL_LINES`/`sponsorGoalLine()`, used in `app_actions.js checkSponsorObjectives()` on goal completion
- Trait evolution / reputation archetypes → `app_race.js checkTraitEvolution()`/`checkNewArchetypes()`, definitions in `data.js` (`TRAIT_POOL`: 11 traits, `REPUTATION_ARCHETYPES`: 9 archetypes). Every trait and every archetype is mechanically real — hooks live across `simulateRace()`/`simulateQualifying()`/`driverStrength()` (`engine.js`) and the leadership morale bump (`app_race.js`). None are purely cosmetic badges.
- Driving Style → `engine.js recordStyleProgress()` sets it (Aggressive/Defensive/Tyre Saver only — other `DRIVING_STYLES` values are AI-grid-only); small real pace/tire-wear effect in `simulateRace()`. `radioLine()` (`data.js`) resolves it through `DRIVING_STYLE_TO_PERSONALITY` before bank lookup.
- NPC/rival `respect` → dampens/accelerates rivalry-level gain and gates the Hot Head trait (`updateRivalry()`/`checkTraitEvolution()`, `app_race.js`); Team Principal `respect` affects turnover timing (`processStaffTurnover()`, `engine.js`).
- NPC `personality` (Calm/Energetic/Strict/Funny) → flavors all 7 roles' `getTalkLinesForRole()` result text (`app_actions.js`) via `npcFlavorLine()`.
- Training programs (13 total, including Fuel Management/Discipline/Leadership) → `app_actions.js renderTrainingModal()` `PROGRAMS` array + `doTraining()`
- Mentor/protégé system → `app_actions.js` (mentor modal/session), `engine.js` (`makeProtege`, `makeGraduatedProtegeDriver`, wired into `processGridRetirements`)
- Car upgrades (per-TEAM, not per-player), facilities, contracts, season objectives display → `app_screens.js renderTeamScreen()`
- Season-end flow (aging, drift, decay, retirement, awards, contracts) → `app_screens.js handleSeasonEnd()`; AI team-car dev math in `engine.js developTeamsForNewSeason()`
- Car setup hints/ideal setup/pace effect → `engine.js` (`trueIdealSetup`, `setupHintConfidence`, `setupMatchBonus`), UI in `app_race.js renderSetupSliders()`, per-track tuning in `data.js` (`idealSetup`)
- Cosmetics → `data.js COSMETIC_POOL` + `app_screens.js renderProfileModal()`
- Year/season display text → `seasonLabel(state)`/`seasonNumToYear()` from `engine.js` — never interpolate `state.season` directly
- Practice/Qualifying mini-games → `data.js QUIZ_QUESTIONS`/`pickQuizQuestions()`, flow in `app_race.js startQuizMiniGame()`/`runPractice()`/`resolveQualifying()`. Every one of the 18 `TRACKS` has at least one question with a `trackId` keyed to that track's real facts (`character`/`overtakeDiff`/`rainChance`); a track's question can never surface at a different track. General (no `trackId`) questions are always eligible and mix in.
- Grid/staff aging/retirement/turnover/scouting offers → `engine.js` (`retirementChanceForAge`, `processGridRetirements`, `processStaffTurnover`, `checkScoutingInterest`), called from `app_screens.js handleSeasonEnd()`
- Romance/dating → `app_actions.js` (`isRomanceEligible`, `getRomanceLinesFor`, `renderNightlifeModal`, `getCurrentPartner`), types in `data.js ROMANCE_TYPES`
- Tutorial/Help content (9 tutorial steps, 20 Help topics) → `app_tutorial.js`
- First-time explainers (6 currently wired — see ARCHITECTURE) → `app_actions.js showFirstTimeExplainer()`, call sites scattered per-feature
- TEAMS<->state persistence → `app_core.js syncTeamsToState()`/`syncTeamsFromState()` — see ARCHITECTURE above
- Colors/fonts/spacing → `styles.css` (`:root` first)
- Boot screen branding → `app_core.js renderBoot()` + `shell.html <title>`
- Season awards mechanical rewards → `data.js SEASON_AWARD_REWARDS`/`seasonAwardReward()`, applied in `app_screens.js handleSeasonEnd()`
- Chief Engineer setup reveal → `app_race.js renderSetupSliders()`, gated on Chief Engineer `relationship.trust>=85`, rolled once per round via `round.chiefEngineerRevealRolled`
- Home Tracks → `engine.js isHomeTrack()`/`HOME_TRACK_THRESHOLD`/`HOME_TRACK_PACE_BONUS`, `driver.isHomeTrackBonus` set in `app_race.js` alongside `familiarityBonus`, badge in `renderRaceCenterScreen()`
- Multi-weekend mechanical-risk chain → `engine.js applyMechanicalRiskAfterDNF()`/`decayMechanicalRisk()`/`clearMechanicalRisk()`, `p.mechanicalRiskMult` read inside `simulateRace()`'s failChance, wired in `app_race.js executeRace()`, cleared via `app_screens.js renderTeamScreen()`'s "Address Reliability Concern" button
- Controversy events → `data.js CONTROVERSY_EVENTS`/`pickControversyEvent()`, rolled in `app_hub.js maybeTriggerControversyEvent()`/`advanceDay()`, shown via `renderControversyModal()`
- Teammate relationship arc → `driver.teammateTally` tracked in `app_race.js executeRace()`, shown in `app_actions.js renderRelationshipsModal()` and captured/reset in `app_screens.js handleSeasonEnd()`
- Constructors' Championship → `app_hub.js constructorsStandings()` (reads `state.constructorPoints`, NOT derived from `s.standings`+current team), points tracked at scoring time in `app_race.js executeRace()`/`resolveSprintRace()`, displayed in `app_screens.js renderTeamScreen()` and `renderSeasonWrapModal()`
- Multi-driver press conference → `app_actions.js renderMediaModal()`'s `presentFromLastRace()` check
- Manager/Agent NPC → 8th role in `app_core.js generateNpcRoster()`, talk lines in `app_actions.js getTalkLinesForRole()`, sponsor-lead flavor in `data.js MANAGER_LEAD_LINES`/`managerLeadLine()`
- Pit-stop strategy (undercut/overcut) → `app_race.js maybeApplyPitStopStrategy()`, selected in `renderRaceStrategyPrompt()`, threaded through `renderFormationLapMoment()`/`executeRace()`
- Qualifying red flag (Q3 deletion) → `app_race.js maybeApplyQualifyingRedFlag()`, called from `resolveQualifying()`
- Multi-season storyline callbacks → `app_race.js checkMultiSeasonCallbacks()`, called from `executeRace()` alongside `checkMilestones()`
- LEGACY_KEY/HOF_KEY rename+migration → `engine.js` (`LEGACY_KEY`/`OLD_LEGACY_KEY`, `HOF_KEY`/`OLD_HOF_KEY`), same pattern as `SAVE_KEY`/`OLD_SAVE_KEY`
- Named driver roster (nationality/age/bio/pre-loaded careerStats per team) → `data.js TEAM_DRIVER_PROFILES` (source of truth; `RIVAL_DRIVER_NAMES` is derived FROM this, don't hand-maintain it separately)
- Manual teammate pick (character creation + mid-career switches) → `app_core.js renderTeammatePickModal()`, called from `renderCreateCharacter()`'s Start Career button and from `app_screens.js renderScoutingOfferModal()`/`renderContractOffersModal()`'s accept handlers on any genuine team change
- Grid generation with an explicit teammate choice → `engine.js generateGrid(playerTeamId, playerTeammateChoice)` returns `{grid, abandonedName}` (NOT a bare array)
- Mid-career team-switch driver-keep choice → `engine.js rebalanceGridForTeamChange(state, oldTeamId, newTeamId, keepDriverId)`
- **(Pass 22)** Weather radar trend strip → `app_hub.js renderRadarTrendStrip()`, called from the same forecast-chip function that already calls `forecastGuess()`
- **(Pass 22)** Paddock vote (season-end flavor comparison) → `app_screens.js computePaddockVote()`, called from `handleSeasonEnd()`, rendered in `renderSeasonWrapModal()`
- **(Pass 22)** Sponsor renewal negotiation → `app_actions.js checkSponsorObjectives()` (queues `s.pendingSponsorRenewals` at the 10th goal instead of auto-converting) + `renderSponsorRenewalModal()`, chained into the race-result advance flow in `app_race.js finishExecuteRace()`
- **(Pass 22)** Track records board → `app_race.js finishExecuteRace()` (writes `s.trackRecords`, reuses `plausibleLapTime()`), displayed in `app_screens.js renderMuseumModal()`
- **(Pass 22)** Home track testimonial → `app_race.js runPracticeQuiz()`'s onComplete, fire-once via `p.milestones` keyed per-track (`home_track_${trackId}`)
- **(Pass 22)** Practice program choices → `app_race.js PRACTICE_PROGRAMS`/`renderPracticeProgramModal()`/`runPracticeQuiz()` — replaces the old always-automatic `pickPracticeQuizTheme()` roll for the practice flow specifically (qualifying's own theme picker is untouched)
- **(Pass 22)** Team orders → `app_race.js maybeDetectTeamOrdersMoment()`/`applyTeamOrdersSwap()`/`renderTeamOrdersModal()`, detected in `executeRace()` right after every other post-scoring event has settled, resolved via a modal BEFORE `finishExecuteRace()` continues
- **(Pass 22)** Podium press pool with multiple journalists → `app_actions.js GUEST_JOURNALISTS`/`pickPressInterviewer()`, called from `renderMediaModal()`; a guest's presence changes question framing only, relationship state always stays on the regular roster Journalist
- **(Pass 22)** Contract release clause → `app_screens.js renderReleaseClauseSection()`/`renderReleaseClauseConfirmModal()`/`releaseClauseCost()`, shown on the Team screen contract panel
- **(Pass 22)** Simulator "what-if" mode → `app_race.js renderSimulatorPreviewModal()`, called from Race Center; UI-only, runs a throwaway `simulateQualifying()` call that never mutates real state
- **(Pass 22)** Multi-year contract roadmap milestones → generated in `app_screens.js renderContractOffersModal()` (`o.roadmapMilestone`, 2+ year offers only), persisted onto `p.contract.roadmapMilestone` at signing, checked in `handleSeasonEnd()`
- **(Pass 22)** Junior series prologue → `app_core.js runJuniorSeriesPrologue()`/`renderJuniorSeriesIntro()`/`runJuniorSeriesRound()`/`finishJuniorSeriesPrologue()`, runs right after `startNewCareer()` creates state, before the hub/chrome is shown
- **(Pass 22)** Team Ownership Mode → `app_core.js` (character-creation toggle, `player.isTeamOwner`), `app_race.js autoResolveOwnerRaceWeekend()`/`this._ownerSilent` (makes `openModal`/`closeModal`/`startQuizMiniGame` no-op/auto-resolve instead of duplicating scoring logic), `app_hub.js renderTodayActions()`'s owner-mode branch
- **(Pass 22)** Weather microclimates → `data.js TRACKS[].microclimate` flag (Spa, Silverstone), `app_race.js maybeApplyWeatherShift(results, weather, track)`'s track-aware fire rate
- **(Pass 22)** Sim racing side career → `app_actions.js doSimRacingActivity()`, `SPONSOR_POOL`'s Circuitworks entry (`needsSimRacingRep:true`) gated via `data.js sponsorUnlocked()` (layers on top of `sponsorTierUnlocked()`, doesn't replace it)
- **(Pass 22)** Driver academy roster → `app_actions.js ACADEMY_MAX_SIZE`/`renderMentorModal()`/`recruitAcademyProspect()`/`mentorProtege()`, `engine.js processGridRetirements()` scans the whole `p.protegeRoster` for a graduated prospect
- **(Pass 22)** Legend exhibition mode → `app_screens.js LEGEND_EXHIBITION_POOL`/`buildLegendExhibitionGrid()`/`renderLegendExhibitionModal()`, reuses `makeLegendDriver()`/`simulateQualifying()`/`simulateRace()` directly; non-persistent, never touches `App.state`

---

## EXTENSION CHECKLIST — READ BEFORE ADDING ANYTHING NEW

1. **Screen or modal?** Screen = persists in `#screen`, needs nav + `.active`
   toggling; most new features should be a modal instead.
2. **Modal**: start its HTML with `this.modalHeader('Title')` — never skip.
3. **Does this modal need to be must-resolve?** If state already moved
   forward before the modal opened, in a way an X-close can't undo, hide the
   close button and disarm the overlay click (see ARCHITECTURE above).
4. **Any driver/NPC name in rendered HTML**: use `this.editableName(id, kind,
   name)`, then `this.wireEditableNames(container)` after inserting
   (`openModal()` already does this for you).
5. **New full screen**: do NOT add nav-click wiring — only toggle
   `.nav-item.active` + call `this.updateStatusStrip()`.
6. Mutate `App.state`, then `this.saveState()` — never call `saveGame()`
   directly (this also skips the TEAMS sync and skill cap — see
   ARCHITECTURE above).
7. **New mutable field on a TEAMS entry?** Add it to BOTH
   `syncTeamsToState()` and `syncTeamsFromState()` (`app_core.js`) or it will
   silently reset on every reload.
8. **New rule that should apply after EVERY state change?** Add it to
   `saveState()` (`app_core.js`), following the skill-cap pattern — don't
   thread it through every individual write site.
9. Stat bars → `this.renderPixelBar(value, colorClass)`, never hand-rolled markup.
10. Reuse existing CSS classes (`.pixel-panel`, `.chip`, `.pbtn` variants,
    `.option-grid`/`.option-card`, `.headline-card`) before inventing new ones.
11. **First-time explainer?** Fire `showFirstTimeExplainer()` synchronously,
    never behind a `setTimeout` — see TESTING WORKFLOW gotcha above.
12. **Guide/help text?** Flowing paragraphs, not `<br>`-line fragments — see
    DESIGN LANGUAGE above.
13. **After the change**: rebuild → syntax check → full test suite (see
    TESTING WORKFLOW). Not done until it's clean.
14. Grep for your new method name across all files before adding it — silent
    overwrites from `Object.assign(App,{...})` collisions don't error.
15. If you touch anything under STATE SHAPE or REMAINING BACKLOG, update
    that entry (mark done / adjust scope / add the new field) so the next
    session isn't stale.

---

## REMAINING BACKLOG

Everything Pass 22's 17 feature ideas covered is DONE (see CHANGELOG for
the list and scope notes). What's genuinely left:

- **Team Ownership Mode's own dedicated screens.** Pass 22 deliberately
  scoped this down to a flag-on-the-existing-save-shape implementation
  (auto-resolved race weekends via `_ownerSilent`, the player's own driver
  still exists) rather than the brief's full "own UI screens and a
  redesigned driverStrength consumer for two AI-controlled seats." A truly
  separate two-seat management screen (hiring/firing two AI drivers,
  strategy calls FOR them, a development-budget-only view with no driver
  stats at all) would need `simulateRace()`'s player-vs-grid distinction
  rebuilt — real, substantial surgery on the core sim, not attempted here.
- **Dedicated automated tests for older Pass 19 mechanics** beyond what
  Pass 22 added: mechanical-risk chain, teammate battle arc, season award
  rewards, controversy events, Manager NPC, multi-season callbacks are
  still only covered indirectly (via regression_test.js's full playthrough
  and scattered assertions elsewhere), not by their own targeted checks.
- **Driver Academy's bench prospects have no individual "arc"** beyond a
  shared latent-progress trickle — the brief mentioned "each with their own
  arc"; what shipped is a shared mechanical trickle, not distinct narrative
  beats per bench prospect. A real per-prospect arc (journal entries,
  personality, a name the player gets attached to before they even start
  training) would be a reasonable next increment.
- **Simulator "what-if" mode only previews qualifying**, not a full race
  outcome, despite the feature idea's framing ("qualifying/race outcome").
  Previewing a full race would need a throwaway `simulateRace()` call too
  (cheap to add — the qualifying preview's pattern generalizes directly)
  but wasn't done to keep the preview screen simple/fast.
- **Legend Exhibition's opponent pool is a fixed 20-name flavor list**
  (`LEGEND_EXHIBITION_POOL`, app_screens.js), not drawn from real
  in-save history (e.g. actual retired grid drivers from THIS save, or
  Hall of Fame entries). A "true greatest grid" pulling from the player's
  own Hall of Fame would be a natural follow-up.

---

## CHANGELOG

- **Pass 1** – Independent AI development team, setup guidance affecting pace, retirement legacy, driver cosmetics.
- **Pass 2** – Year progression UI, Practice/Qualifying mini-games, NPC aging and retirement, expanded retirement careers, Free Time recovery, romance system.
- **Pass 3** – Rotating circuits, FIA regulation changes, sprint weekends, scalable season objectives, championship standings, richer rivals, media improvements, season calendar.
- **Pass 4** – Fixed grid size to 22 drivers and expanded rival grid.
- **Pass 5** – Fixed team-switch grid rebalance, enforced 22-driver limit, UI fixes, dynamic weekly weather.
- **Pass 6** – Major gameplay expansion: race weekend depth, economy overhaul, team development, sponsor progression, difficulty-based career starts, expanded romance, celebrity events, contract system, balancing and bug fixes.
- **Pass 7** – Smarter season objectives after team changes, career milestones, improved engineer feedback, morale affecting contract renewals.
- **Pass 8** – Calendar UI fixes, facility UI improvements, richer rival interactions, unique driver generation, gameplay balancing.
- **Pass 9** – Test suite consolidation and stability improvements.
- **Pass 10** – Fixed calendar progression desynchronization, mandatory event resolution, track-specific setup knowledge, weekly training cap.
- **Pass 11** – Major race simulation expansion: mentor/protégé system, tire strategy, safety cars, penalties, sprint qualifying improvements, weather events, immersion additions, bug fixes.
- **Pass 12** – Gameplay stat audit and rebalance, expanded trait system, improved stat explanations, race modifier display, documentation improvements.
- **Pass 13** – Combined gameplay-flow/UI-UX pass and tutorial/onboarding audit: welcome tour now covers tire compounds, sprint weekends, and mentor/protégé; added Sprint Weekends and Mentor a Protégé to the Help reference; added first-time explainers for sponsor signing and protégé recruitment; tire compound tradeoffs promoted from hover-only tooltips to visible text.
- **Pass 14** – Simplified every tutorial/help/first-time-explainer text into short scannable lines (later reverted — see Pass 16).
- **Pass 15** – Gameplay expansion: fastest laps, rivalry origins, sponsor dialogue, new training programs, save migration, cleanup.
- **Pass 16** – Activated all remaining traits, expanded track-specific quizzes, restored tutorial formatting.
- **Pass 17** – Gave previously cosmetic systems real gameplay impact (respect, driving style, personalities, team culture, reputation archetypes, milestones).
- **Pass 18** – Anti-overpower balance system (dynamic skill cap, AI progression, seasonal stat decay), Skill Ceiling help, documentation audit/rewrite.
- **Pass 19** – Completed remaining gameplay backlog: season awards, Home Tracks, Constructors' Championship, teammate rivalry, manager role, pit strategy, controversy events, reliability chain, multi-driver press, qualifying red flags, multi-season callbacks, expanded tracks/question pool, balancing, bug fixes, verification.
- **Pass 20** – Replaced generic roster with named teams/drivers, teammate selection, AI career histories, nationality expansion, grid-generation update, compatibility and test fixes.
- **Pass 21** – Full audit pass: major gameplay bug fixes, dead-code cleanup, Help expansion, UI audit, test consolidation.
- **Pass 22** – Combined pass: 17 new feature ideas
  core) — every flaky check was individually verified to pass in isolation.
  See TESTING WORKFLOW's gotcha note below for the diagnostic pattern and
  the mitigation already applied (splitting suite_features_test.js in two).
