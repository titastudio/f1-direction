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
   for t in regression_test coverage_audit retirement_test suite_economy_test suite_progression_test suite_features_test; do
     timeout 120 node $t.js 2>&1 | tail -5
   done
   ```
   Confirm everything is clean BEFORE making changes, so any later failure is
   known to be yours, not a pre-existing issue. Current clean-baseline check
   counts: `suite_economy_test.js` 23, `suite_progression_test.js` 9,
   `suite_features_test.js` 28 (plus the 3 structurally-separate scripts).
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
Mitigations already in the test suite:
- Wrap long loops in try/catch, treat those four error strings as a sandbox
  crash (report separately, don't fail the whole test on it).
- Keep any single playthrough's target modest (regression_test.js targets 5
  rounds, ~85 iterations) — split into multiple `puppeteer.launch()` calls or
  fast-forward state directly via `page.evaluate()` (see retirement_test.js,
  which jumps `calendar.currentRound` to the season's last round instead of
  clicking through ~200 iterations) rather than raising the budget and hoping.
- Launch with `args:['--no-sandbox','--disable-dev-shm-usage','--disable-gpu','--disable-extensions']`.

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

### Test suite (6 runnable scripts + 1 shared helper module)

```
_test_helpers.js         NOT a test -- shared launch/robustClick/startNewCareer/
                          runCheck/printSummaryAndExit helpers, required by the
                          suite_*.js files below so they don't each repeat the
                          same boilerplate.

regression_test.js       full playthrough: new career, sponsor sign, race loop, zero console errors
coverage_audit.js        4 viewports (360×640, 375×667, 390×844, 1440×900) × every screen/modal,
                          checks no overflow under #bottom-nav, no horizontal scroll, close button reachable
retirement_test.js       force-age retirement flow → legacy perk → RETIRES screen

suite_economy_test.js    23 checks -- car development, facility effects, training/setup-confidence
                          pacing, performance-clause contracts, sponsor rework, difficulty knobs,
                          stat/archetype wiring (mood, mediaReputation sponsor unlock, Tyre
                          Whisperer DNF relief)

suite_progression_test.js 9 checks -- celebrity encounters, romance overhaul, multi-career Hall
                          of Fame, legacy-perk legend-driver carryover, difficulty-gated starting
                          teams, mentor/protégé graduation, trait-evolution fixes (Ice Cold/Late
                          Braker/Workhorse/Hot Head, plus a false-positive guard)

suite_features_test.js   28 checks -- race-weekend depth features, the must-resolve-modal fixes,
                          22-driver grid invariant, sprint weekend flow, stat-attributed radio
                          lines, weather-shift events, "current pace modifiers" chip
```

Run each with a `timeout` wrapper — 6 commands total instead of running every
check file individually:
```bash
for t in regression_test coverage_audit retirement_test suite_economy_test suite_progression_test suite_features_test; do
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
counts in this section when you do.**

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
    sponsors:[], sponsorHistory:[{name,season}], unlockedSponsorTiers:[1],
    milestones:[], journal:[{season,week,title,text,icon}],
    seasonAwardsWon:[{season,award}], retired:false,
    difficulty:'Casual'|'Standard'|'Realistic',
    setupTrainingByTrack:{[trackId]:number},  // per-track sim training count, feeds setupHintConfidence -- NOT global
    trainingActionsThisWeek:number, trainingWeekTracker:number|undefined,  // weekly training cap (3/week), tracker resets against s.week
    recentQuizIds:[ids],
    protege: null|{name,nationality,flag,mentorProgress:0-100,graduated:boolean},
      // recruit at homeLevel>=3; graduates ONLY into a real grid retirement
      // slot via processGridRetirements(), never as an extra 23rd driver
    joinedTeamOnSeason: number,
      // season the player joined their CURRENT team; a real team switch
      // resets this and costs morale (team-switch cooldown), a same-team
      // contract renewal does not
    seenStatEffectExplainer, seenMentorExplainer, seenRomanceExplainer,
    seenSponsorExplainer, seenDevSlotsExplainer, seenCelebrityExplainer: boolean|undefined,
      // one flag per first-time-explainer -- see ARCHITECTURE above for the
      // full current list; grep `showFirstTimeExplainer(` to double check
  },
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

## REMAINING BACKLOG (not built yet)

Open-ended feature ideas — each deserves its own scoping conversation (what
mechanic, why, how it interacts with existing systems) before any code.
There are currently no small/independently-scoped "gaps to fix" outstanding;
everything previously listed there has been closed (see CHANGELOG).

- Season awards (`computeSeasonAwards()`, `app_screens.js`) getting a real
  mechanical hook instead of badge-only, mirroring the archetype/trait
  hook pattern already used repeatedly elsewhere
- NPC roster staff involvement in race weekend itself (e.g. a high-trust
  Chief Engineer occasionally reveals the ideal setup outright)
- Track familiarity surfaced as its own "home tracks" concept, not just a
  hidden setup-hint input
- A multi-weekend consequence chain for mechanical failures (this
  weekend's DNF slightly raises next weekend's risk unless addressed)
- Public-perception moments that swing fanPopularity/mediaReputation in
  opposite directions based on a real stance — **partially already built**:
  `renderMediaModal()` (`app_actions.js`) already has stance-based branches
  that swing team morale/reputation in different directions (e.g. "Blame
  the car" hurts morale, "Fire back with confidence" escalates rivalry
  instead of just giving a uniform plus). Remaining gap is narrower than it
  sounds: a dedicated one-off "controversy" event type distinct from the
  existing race-result/rivalry press questions, if that's still wanted.
- Teammate relationship arc / season-long head-to-head storyline
- Constructors' championship (mostly a display layer over `TEAMS[].
  carStrength`/`reliability`, which already exist)
- Post-race press conference as a multi-driver moment, not always 1-on-1
  (`renderMediaModal()`, `app_actions.js` is currently player-only) —
  rival/teammate presence in the room changing which options appear
- A Manager/Agent NPC role (the 7 fixed `generateNpcRoster()` roles,
  `app_core.js`, don't include anyone who negotiates on the player's
  behalf) — occasional contract pushback or a sponsor lead the player
  wouldn't otherwise see
- Weekend-long tire/fuel strategy instead of one single choice — a mid-
  race pit-stop-timing decision (undercut/overcut), resolved the same
  presentational post-scoring way the safety car and grid penalty events
  already are (`simulateRace()`, `engine.js`), so it wouldn't touch core
  sim math
- A qualifying "moment" to match the race's — Q1/Q2/Q3 elimination
  framing is already cosmetic (`app_race.js`, the `expectedSegment`
  labeling), so a small-chance red-flag/track-limits deletion event in
  Q3, same post-scoring pattern as the safety car, would give qualifying
  its own drama beat
- Multi-season storyline callbacks — milestones/journal entries are all
  one-off; a thread that resurfaces 2-3 seasons later (a mentored
  protégé becoming a rival, a team you left resenting you) would reuse
  `history[]` and the existing protégé/legend-driver machinery
  (`makeProtege`/`makeLegendDriver`, `engine.js`) rather than needing new
  state
- SAVE_KEY's sibling keys (`LEGACY_KEY`/`HOF_KEY`, both still
  `'apexpal_...'`) could get the same rename-with-migration treatment
  `SAVE_KEY` got in Pass 15, if wanted — deliberately left alone so far
  since it wasn't asked for and touches two separate persistence paths

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
- **Pass 15** – Closed six standing gaps together: weighted fastest-lap roll (`rollFastestLap()`) feeding `careerStats.fastestLaps` + a small bonus + result-modal banner; removed the dead `state.player.relationships` field; added rivalry-origin flavor text (`RIVALRY_ORIGINS`/`rollRivalryOrigin()`); added sponsor type-specific goal-completion dialogue (`SPONSOR_GOAL_LINES`/`sponsorGoalLine()`); added three new training programs giving `fuelSaving`/`discipline`/`leadership` a real write path; renamed `SAVE_KEY` to `'formulaonedirection_save_v1'` with a migration fallback from the old key.
- **Pass 16** – Triggered the 5 remaining `TRAIT_POOL` traits (Lucky, Leader, Fragile, Bad Starter, Overconfident), each with a small real mechanical hook — all 11 traits now mechanically real. Added track-specific quiz questions for all 18 `TRACKS` via a new `trackId` field, filtered so a track's question can never surface elsewhere. Reverted Pass 14's `<br>`-line-fragment text format back to flowing paragraphs — same content, formatting only.
- **Pass 17** – Fixed six shallow/cosmetic-only systems flagged by a full-codebase audit: `respect` (previously write-only) now affects rivalry-gain rate, gates Hot Head, and affects Team Principal turnover timing; Driving Style (previously 100% cosmetic) now has a small real pace/tire-wear effect, and `radioLine()` no longer silently drops style flavor into the wrong bank; NPC `personality` now flavors all 7 roles' dialogue (previously Race-Engineer-only); team `culture` flavor text is now backed by `devTrend.volatility` scaling morale-swing magnitude; all 9 `REPUTATION_ARCHETYPES` (5 were pure badges, plus a dead `veteran` archetype) now have real hooks; `careerStats.fastestLaps`/`wetWins` now feed milestones.
- **Pass 18** – Anti-overpower balance system, three linked mechanics: (1) player driving/mental stats are capped at the current points-standings leader's same stat + 8, re-evaluated live on every `saveState()` via the new `enforcePlayerSkillCap()` (`app_core.js`); (2) AI grid drivers now drift season-over-season (`developGridDriversForNewSeason()`, `engine.js`) biased by age, so the points leader — and therefore the cap — genuinely shifts identity over a career instead of staying fixed; (3) player stats fade by 2 points every season-end (`handleSeasonEnd()`, `app_screens.js`), smaller than a typical training gain so regular training more than offsets it. Added a "Skill Ceiling" Help topic and a near-cap indicator in the Training modal. Also did a full rewrite of this document: removed stale claims (explainer count, "as of Pass N" changelog-in-quick-reference phrasing), verified every count (11 traits, 9 archetypes, 18 tracks, 11 teams, 10 sponsors, 13 training programs, 9 tutorial steps, 20 Help topics) directly against code, and folded the two prior standalone "Documentation Audit"/"Documentation Reorganization" changelog entries into this rewrite rather than keeping them as separate historical lines with nothing left to point to.
