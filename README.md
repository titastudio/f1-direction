# FORMULA ONE DIRECTION — Project Handoff Document

A single-file HTML/CSS/JS racing career life-sim, pixel/Tamagotchi aesthetic.
Written so a fresh session can start working without re-reading the codebase.

Current deliverable: `formula-one-direction.html` in `/mnt/user-data/outputs/`.

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
   known to be yours, not a pre-existing issue.
3. **Read Section "ARCHITECTURE"** below — three of its patterns
   (nav wiring, modal structure, editable names) caused real shipped bugs
   before and will bite you again if skipped.
4. **Pick your task** from "REMAINING BACKLOG" below, or scope a new one.
5. **After every change**: rebuild → syntax check → full test suite again.
   Add a test for whatever you built (see EXTENSION CHECKLIST). Don't call a
   change done until the suite is clean.

---

## HOW THE PROJECT IS ASSEMBLED

```
shell.html      — HTML skeleton with {{STYLES}} / {{SCRIPTS}} placeholders
styles.css      — all CSS (design tokens, pixel UI components, layout)
data.js         — static game data (teams, tracks, sponsors, name pools, radio lines)
engine.js       — pure logic: driver generation, race math, save/load, constants
app_core.js     — App singleton, boot screen, character creation, editable-name system
app_actions.js  — modal system + Training/Lifestyle/Relationships/Media/Sponsors modals
app_hub.js      — Home screen (the Tamagotchi "pod" hub) + day/week loop
app_race.js     — Race Center screen + practice/qualifying/race sim + results
app_screens.js  — Team&Car, Career Stats, Journal, Museum, Save Menu, Season End,
                  Contract Offers, Retirement
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
Chrome has disconnected anywhere from ~100 to ~560 rapid-DOM-replacement
iterations into a single browser session (varies run to run — a
Puppeteer/Chrome sandbox limitation, not a game bug). Mitigations already in
the test suite:
- Wrap long loops in try/catch, treat `Target closed`/`detached Frame`/
  `Session closed`/`Execution context was destroyed`/`Protocol error` as a
  sandbox crash (report separately, don't fail the whole test on it).
- Keep any single playthrough's target modest (regression_test.js targets 5
  rounds, ~85 iterations) — split into multiple `puppeteer.launch()` calls or
  fast-forward state directly via `page.evaluate()` (see retirement_test.js,
  which jumps `calendar.currentRound` to the season's last round instead of
  clicking through ~200 iterations) rather than raising the budget and hoping.
- Launch with `args:['--no-sandbox','--disable-dev-shm-usage','--disable-gpu','--disable-extensions']`.

**Gotcha — statistical checks need enough trials, and must exclude
sentinel/outlier values from averages.** A DNF's `raceScore` is a `-9999`
sentinel; even a ~5% DNF rate will swing (and can flip the sign of) a
few-hundred-trial average of a ~15-point true signal. When averaging a
probabilistic effect, filter out DNFs/other sentinels first unless DNF rate
itself is what you're measuring.

### Test suite (6 runnable scripts + 1 shared helper module)

Individual per-feature test files used to number 19 and were annoying to run
one by one during iteration. They're now consolidated into 3 themed
combined suites (each still one browser launch, but many independent check
blocks inside — one failing block doesn't abort the rest, see `runCheck()`
in `_test_helpers.js`), plus the 3 tests that are structurally different
enough to stay separate:

```
_test_helpers.js         NOT a test -- shared launch/robustClick/startNewCareer/
                          runCheck/printSummaryAndExit helpers, required by the
                          suite_*.js files below so they don't each repeat the
                          same boilerplate.

regression_test.js       full playthrough: new career, sponsor sign, race loop, zero console errors
coverage_audit.js        4 viewports (360×640, 375×667, 390×844, 1440×900) × every screen/modal,
                          checks no overflow under #bottom-nav, no horizontal scroll, close button reachable
retirement_test.js       force-age retirement flow → legacy perk → RETIRES screen

suite_economy_test.js    14 checks: per-team car development (tier seeding, dev-slot cap,
                          engineering facility, reload persistence), media/hospitality facility
                          effects, tyre-management training, performance-clause contracts,
                          sponsor rework (10-goal progression, loyalty raise, team-switch
                          survival), all 7 previously-dead stat/difficulty knobs

suite_progression_test.js 7 checks: celebrity encounters (rarity gating, embrace-vs-lowkey),
                          romance overhaul (3-candidate picker, dating perk), multi-career Hall
                          of Fame, legacy-perk legend-driver carryover, difficulty-gated
                          starting teams

suite_features_test.js   14 checks: race-weekend depth features (overtake/forecast chips,
                          podium interview), the "stuck on an off week" season-end stall fix,
                          22-driver grid invariant across both team-switch paths, the full
                          Section 19 backlog pass (objectives rescale, milestones, forecast
                          callback, race engineer advice, morale-affected offers, chief
                          engineer suggestion), and the player-feedback fixes pass (grid
                          name-uniqueness, DNF-risk difficulty scaling, car-upgrade
                          diminishing returns, expanded rival talk options, "This Week"
                          banner honesty)
```

Run each with a `timeout` wrapper — 6 commands total instead of 19:
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
push to `allErrors`) rather than inventing a new pattern. If a suite file is
missing, recreate it from the descriptions above — they're scaffolding, not
precious, same as before.

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

---

## ARCHITECTURE — LOAD-BEARING PATTERNS

### The App singleton
Everything hangs off global `App` (`app_core.js`). Every file does
`Object.assign(App, {...methods...})`. Method names must be unique across
files — a collision silently overwrites with no error. `App.state` holds the
whole save-game.

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

### Universal editable-name system
`this.editableName(id, kind, currentName)` (`kind`: `'player'|'driver'|'npc'`)
renders a rename-on-click span. After injecting HTML containing one, call
`this.wireEditableNames(container)` (defaults to `document`) or the pencil
icon is inert. `openModal()` already does this automatically.

### Save/load
`localStorage` key `'apexpal_save_v1'` (still says "apexpal" from before the
rename — intentionally left alone to avoid a breaking migration).
`saveGame()`/`loadGame()`/`deleteSave()` are plain functions in `engine.js`.
Always call `this.saveState()` after mutating `App.state` (wraps `saveGame`,
keeps future hooks centralized) — never call `saveGame()` directly.

### Rebuild-safe editing
Always `view` the exact current file section immediately before editing with
`str_replace`-style tools. Stale/remembered content has previously deleted a
needed function or left orphaned duplicate code — both broke the game and
were only caught by the regression test, not syntax checking. **Syntax check
is necessary but not sufficient — always also run the regression test.**

---

## STATE SHAPE (`App.state`)

```js
{
  player: {
    id:'player', name, gender, nationality, flag, age, teamId,
    attributes: {
      driving: { racePace, qualifyingPace, cornering, braking, starts,
                 overtaking, defending, wetSkill, tyreManagement,
                 fuelSaving, consistency, reaction },   // 0-99
      mental:  { confidence, focus, pressure, leadership, discipline, adaptability }, // 0-99
      lifestyle: { fitness, mood, stress, energy },      // 0-100
    },
    fanPopularity, mediaReputation, traits:[], reputationArchetypes:[],
    drivingStyle: null|string, styleProgress:{},
    contract:{teamId,years,salary}, money, homeLevel:1-6,
    careerStats:{starts,wins,podiums,poles,fastestLaps,dnfs,points,wetWins},
    trackFamiliarity:{[trackId]:65-100}, sponsors:[], sponsorHistory:[{name,season}],
    milestones:[], journal:[{season,week,title,text,icon}],
    seasonAwardsWon:[{season,award}], retired:false,
    difficulty:'Casual'|'Standard'|'Realistic',
    setupTrainingByTrack:{[trackId]:number},  // per-track sim training count, feeds setupHintConfidence -- NOT global
    trainingActionsThisWeek:number, trainingWeekTracker:number|undefined,  // weekly training cap (3/week), tracker resets against s.week
    recentQuizIds:[ids],
  },
  grid: [ /* ~21 AI drivers + player = 22 total, see makeRivalDriver() in engine.js */
    { id,name,teamId,age,nationality,flag,drivingStyle,gender:'M',
      attributes:{...}, fanPopularity, mediaReputation, traits:[],
      careerStats:{...}, memories:[], rivalryLevel:0-100, respect:50,
      isLegend: true|undefined, romance:{status,spark} }
  ],
  season:number,                 // opaque internal counter, starts 2027 — NEVER shown to player
  careerStartSeason:number,      // use seasonLabel()/seasonNumToYear() for display, never raw `season`
  calendar:{ rounds:[{round,trackId,completed,isSprint,weather,trackTemp}], currentRound:index },
  dayIndex:0-6, week:number, standings:{[driverId]:points},
  newsFeed:['string w/ emoji'], seasonStats:{wins,podiums,points,beatTeammate,q3count},
  seasonObjectives:[{type,text,target}],   // regenerated each season AND on any team switch
  npcRoster: [ {id,role,name,gender,tenureSeasons,relationship:{friendship,trust,affection,respect,compatibility,memories},
                personality,romance} ],   // exactly 7 fixed roles
  careerRetirementCount:number, metPartner:null|{...}, history:[{season,team,standing,points}],
  rivalId:null|driverId, carUpgrades:{aero,engine,chassis,suspension,reliability}, // each 1-10
  facilities:{gym,simulator,engineering,factory,media,hospitality},  // each 1-5
  currentSetup:{downforce,balance,gearRatio}, tutorialSeen:false,
  teamMorale:{[teamId]:0-100}, lastQualyResults, lastWeather, prevTeamStrength,
  pendingTierOffer:true|undefined,
  cosmetics:{careerStartSeason,equipped:{helmet,livery,suit,gloves},
             totalRoundsCompletedThisSeason, lastKnownUnlocked:[ids]},
}
```

**Sponsor object** (in `player.sponsors[]`): `{name,type,trait,pay,
needsClean/needsAggression/needsPopularity, objectiveText, objectiveDone,
signedSeason, _attackCount/_streak}`.

**Legacy record** (`localStorage['apexpal_legacy_v1']`, NOT in `App.state`,
survives `deleteSave()`): `{perkId, retiredPlayerSnapshot}`.

---

## QUICK-REFERENCE: FILE → WHAT LIVES THERE

- Team/track/sponsor/name-pool data → `data.js`
- Race-result math / points / difficulty balance → `engine.js`
- Character creation fields → `app_core.js renderCreateCharacter()` + `createPlayer()` in `engine.js`
- New day-action → `app_hub.js renderTodayActions()` + `app_actions.js openActionModal()` + a new `render*Modal()`
- Race weekend flow (practice/qualifying/race/strategy) → `app_race.js`
- Car upgrades, facilities, contracts, season objectives display → `app_screens.js renderTeamScreen()`
- Season-end flow, awards, retirement, contract offers, legacy perks → `app_screens.js` (bottom half); AI dev math in `engine.js developTeamsForNewSeason()`
- Car setup hints/ideal setup/pace effect → `engine.js` (`trueIdealSetup`, `setupHintConfidence`, `setupMatchBonus`), UI in `app_race.js renderSetupSliders()`, per-track tuning in `data.js` (`idealSetup`)
- Cosmetics → `data.js COSMETIC_POOL` + `app_screens.js renderProfileModal()`
- Year/season display text → `seasonLabel(state)`/`seasonNumToYear()` from `engine.js` — never interpolate `state.season` directly
- Practice/Qualifying mini-games → `data.js QUIZ_QUESTIONS`/`pickQuizQuestions()`, flow in `app_race.js startQuizMiniGame()`/`runPractice()`/`resolveQualifying()`
- Grid/staff aging/retirement/turnover → `engine.js` (`retirementChanceForAge`, `processGridRetirements`, `processStaffTurnover`), called from `app_screens.js handleSeasonEnd()`
- Romance/dating → `app_actions.js` (`isRomanceEligible`, `getRomanceLinesFor`, `renderNightlifeModal`, `getCurrentPartner`)
- Tutorial/Help content → `app_tutorial.js`
- Colors/fonts/spacing → `styles.css` (`:root` first)
- Boot screen branding → `app_core.js renderBoot()` + `shell.html <title>`

---

## EXTENSION CHECKLIST — READ BEFORE ADDING ANYTHING NEW

1. **Screen or modal?** Screen = persists in `#screen`, needs nav + `.active`
   toggling; most new features should be a modal instead.
2. **Modal**: start its HTML with `this.modalHeader('Title')` — never skip.
3. **Any driver/NPC name in rendered HTML**: use `this.editableName(id, kind,
   name)`, then `this.wireEditableNames(container)` after inserting
   (`openModal()` already does this for you).
4. **New full screen**: do NOT add nav-click wiring — only toggle
   `.nav-item.active` + call `this.updateStatusStrip()`.
5. Mutate `App.state`, then `this.saveState()` — never call `saveGame()` directly.
6. Stat bars → `this.renderPixelBar(value, colorClass)`, never hand-rolled markup.
7. Reuse existing CSS classes (`.pixel-panel`, `.chip`, `.pbtn` variants,
   `.option-grid`/`.option-card`, `.headline-card`) before inventing new ones.
8. **After the change**: rebuild → syntax check → full test suite (see
   TESTING WORKFLOW). Not done until it's clean.
9. Grep for your new method name across all files before adding it — silent
   overwrites from `Object.assign(App,{...})` collisions don't error.
10. If you touch anything under KNOWN GAPS or REMAINING BACKLOG below, update
    that entry (mark done / adjust scope) so the next session isn't stale.

---

## KNOWN GAPS (honest stubs, good starting points for depth)

1. Post-retirement "Manage a Driver Academy"/"Become a Team Principal" are
   stub buttons showing a toast only.
2. `TRAIT_POOL` in `data.js` lists 11 traits; only 3 are ever actually
   awarded (Nervous Under Pressure, Composed Veteran, Fan Favorite). The rest
   are defined but never triggered.
3. `state.player.relationships` is declared but unused — real relationship
   data lives on `npcRoster[i].relationship` / `grid[i].rivalryLevel`.
4. `careerStats.fastestLaps` is tracked in the schema but never incremented
   for anyone — no fastest-lap roll exists in `simulateRace`.
5. Rivalry is purely numeric (closeness of finishing position + decay) — no
   rivalry-origin flavor text.
6. Sponsor "type" (Luxury Brand/Tech Company/etc.) drives objective selection
   only — no type-specific dialogue or events.
7. No sound/music, no true pixel-art sprites (emoji + CSS borders/shadows
   instead) — both deliberate scope decisions, not oversights.
8. `SAVE_KEY` still says `'apexpal_save_v1'` — cosmetic leftover from the
   pre-rename name, intentionally unchanged to avoid a breaking save migration.

---

## CHANGELOG (one line per prior pass — ask if you need the full history)

- **Pass 1**: independent AI team dev, car setup hints w/ real pace effect,
  retirement legacy system, cosmetics (helmet/livery/suit/gloves).
- **Pass 2**: in-game "Year N" labeling, Practice/Qualifying quiz mini-games,
  dynamic NPC/rival aging+retirement, expanded post-retirement options,
  Free Time energy recovery, romance system v1.
- **Pass 3**: rotating track pool, FIA regulation changes, sprint weekends,
  season objectives pay real rewards, full championship standings, richer
  rival profiles, contextual media interviews, Season Calendar screen.
- **Pass 4**: fixed grid being 23 drivers instead of 22; expanded Rival Grid UI.
- **Pass 5**: fixed mid-career team switch not rebalancing the grid; hard
  22-driver cap; fixed header/status-strip overlap; weekly weather/temp system.
- **Pass 6**: race-weekend depth (overtake-tier chip, forecast chip, rival
  radio, podium interviews), economy rework (training/car/facility costs),
  per-team car development + dev slots, sponsor rework (10-goal progression,
  permanent partners), difficulty-gated starting teams, romance overhaul
  (3 types + passive perks), celebrity encounters, wired up 7 previously-dead
  stats/knobs, multi-career Hall of Fame, performance-clause contracts;
  **critical fixes**: stuck-on-an-off-week-forever bug, reject/retry infinite loops.
- **Pass 7**: fixed season objectives not rescaling on a
  mid-season team switch; added milestones for 100 starts/10 wins/5
  poles/250+500 points; forecast-accuracy callback; Race Engineer now gives
  situationally-aware advice (weather/temp/grid-position) instead of a
  hardcoded line; team morale now nudges the current-team contract renewal
  offer; Chief Engineer can suggest the next development priority (talk-menu
  option + persistent Team screen hint). Also hardened `regression_test.js`/
  `retirement_test.js` against this sandbox's headless-Chrome disconnects and
  fixed a flaky DNF-sentinel-skewed average in `dead_stats_test.js`. New test:
  `backlog19_test.js`. Full 18-script suite + all 4 coverage-audit viewports
  clean at the end of this pass.
- **Pass 8** (this pass, player-reported issues): **confirmed NOT a bug** --
  closing the race result modal only ever advances one day; the "skips a
  week" feeling was the hub's "This Week" banner instantly showing the NEXT
  round's track name the moment the race weekend ended, even with several
  free days still ahead. Fixed by making the banner say "Free — next race in
  N days" until the weekend actually starts, plus a toast on the transition
  (`renderHub()`/`executeRace()` in `app_hub.js`/`app_race.js`).
  **Confirmed working correctly, made visible**: all 6 facilities (gym,
  simulator, engineering, factory, media, hospitality) already had real
  mechanical effects -- `facilityEffectText()` (`app_screens.js`) now shows
  the actual current-level bonus number instead of static flavor text, so
  players can verify it. Caught and fixed a real bug while doing this: the
  media facility's displayed cap said "+45% at Lv.5" (stale comment) when the
  formula actually gives +60% -- corrected in both the display text and the
  original comment in `app_actions.js`.
  **Fixed real bug**: rival talk (`renderRivalTalkModal()`, `app_actions.js`)
  only ever offered 2 fixed generic options for every rival, forever --
  meaningful for male players especially since rivals are all-male and
  romance lines never applied. Added a "study their driving style" option
  (small real Adaptability effect) plus situational options for teammates,
  high-rivalry/current-rival, and established head-to-head records.
  **Fixed real bug (confirmed, reproducible from season 0)**: grid driver
  name duplicates. Each team's `RIVAL_DRIVER_NAMES` pool only has 2 names,
  both normally already in use by that team's own starting drivers, so every
  retirement/team-switch replacement's old name-picking logic (checked only
  one team's pool, or excluded only the retiring driver's own name) produced
  duplicates almost immediately. Added `generateFreshDriverName()`
  (`engine.js`) -- checks uniqueness against the ENTIRE grid + player, with a
  484-combination cross-combined fallback pool (`FALLBACK_FIRST_NAMES`/
  `FALLBACK_LAST_NAMES` in `data.js`) replacing the old un-checked
  `"${nationality} Rookie"` placeholder. Verified with a 100-forced-season
  stress test: zero duplicates (previously: duplicates in every single
  season, including season 0).
  **Balance changes (addressing "too easy to stay dominant")**: (1) manual
  car upgrades now have diminishing carStrength gains the further a team
  already is above the grid average (full +1 up to average, tapering to a
  +0.25 floor by +25 over) -- previously a maxed Engineering facility gave a
  flat +1/upgrade x up to 7 slots/season forever with nothing pulling a
  dominant team back down, unlike the AI-side rubber-band in
  `developTeamsForNewSeason()`. (2) DNF/reliability risk now scales by
  difficulty via new `dnfExponent`/`dnfMult`/`dnfCap` fields in
  `DIFFICULTY_SETTINGS` (`engine.js`) -- Realistic now meaningfully punishes
  a weak car (was ~4.5% max DNF chance, now ~8-10%) while a strong car stays
  safe either way (~0.1-0.5%), widening the risk gap between grid tiers
  instead of scaling every tier by the same flat ratio. New test:
  `feedback_fixes_test.js` (name-uniqueness stress test, DNF-scaling
  statistical check, upgrade diminishing-returns check, rival talk option
  count, This-Week banner honesty). Full 19-script suite + all 4
  coverage-audit viewports clean at the end of this pass.
- **Pass 9** (test-only, no game logic touched): consolidated the test
  suite from 19 individual files down to 6 runnable scripts (3 that stayed
  separate -- `regression_test.js`, `coverage_audit.js`, `retirement_test.js`
  -- plus 3 new combined `suite_*.js` files covering the other 16 former
  files' checks as independent blocks in one browser session each: see
  TESTING WORKFLOW below for the full breakdown). Added `_test_helpers.js`
  (shared launch/robustClick/startNewCareer/runCheck/printSummaryAndExit
  helpers) so the suite files don't each repeat the same boilerplate. Also
  fixed a flaky check while porting it over: the team-morale-vs-contract-
  offer comparison (Section 19.6) compared one noisy sample per side, and
  the offer's own `randInt(0,2)` salary roll could invert the comparison
  ~11% of the time even though the underlying morale effect is real -- now
  averages 20 samples per side, same fix shape as the earlier DNF-sentinel
  flakiness fix in dead_stats_test.js. Full 6-script suite clean (4
  consecutive runs) at the end of this pass.
- **Pass 10** (critical bug fixes + design changes, from a real playtest +
  full audit of every `wireModalClose()` call site): **critical, confirmed
  reproducible bug**: the race/qualifying/sprint result modals' X close
  button bypassed `advanceDay()` entirely (`closeModal()` just removes the
  DOM node, no re-render) while `round.completed`/`s.calendar.currentRound`
  had ALREADY been advanced before the modal even opened. This desynced
  `s.dayIndex` from the calendar and left a STALE hub underneath still
  showing the just-finished track's race button — clicking it then raced
  the NEXT track under the OLD track's banner (exactly the "Suzuka shows
  again, click it, actually races Silverstone" report). Fixed by making all
  three result modals must-resolve (X hidden, overlay disarmed), same
  pattern as the podium interview/quiz/season-wrap modals. Audited all 28
  `wireModalClose()` call sites in the codebase for the same class of bug —
  found and fixed one more: **Contract Offers** (reached when
  `p.contract.years<=0`) let a player X-close it and then race indefinitely
  under no valid contract at all (confirmed reproducible, not just
  theoretical) — also made must-resolve. Every other call site confirmed
  safe (either a pure browsing/informational modal, or an explicit
  accept/decline choice where nothing is pre-committed before the modal
  opens). New regression tests in `suite_features_test.js` guard all 4 fixes
  specifically (`critical: ...` named checks).
  **Design fixes** (player feedback: training/setup progression had no real
  pacing): (1) `setupTrainingByTrack` replaces the old global lifetime
  `setupTrainingSessions` counter (`engine.js` `setupHintConfidence()`,
  `app_actions.js` `doTraining()`) — training now only builds setup
  confidence for the SPECIFIC track it was done at, closing the exploit
  where one early grind session permanently maxed the confidence floor on
  every future track including ones never visited. (2) Added a genuine
  weekly training cap (3 sessions/week, tracked via `p.trainingActionsThisWeek`
  + `p.trainingWeekTracker` against `s.week`, `app_actions.js`) — previously
  the only limiter was energy, which let a player run 5-8 sessions in a
  single day and fully recharge within ~36 hours, no real weekly pacing at
  all. Training modal now shows the weekly cap status and disables all
  programs once reached. New regression tests in `suite_economy_test.js`
  (`training_cap:`/`setup_confidence:` named checks).
  **Investigated, confirmed working as designed, no change needed**: all 6
  facilities' effects (re-verified from Pass 8); adaptability's real but
  narrow effect (`driverStrength()` cold-track grip-penalty relief only —
  the player saw the "+1" toast on a non-cold-track day and couldn't see an
  effect, which is expected, not a bug, given the mechanic's actual scope).
  Full 6-script suite (now 20+18+7+16 = many individual checks) clean at the
  end of this pass, plus a dedicated exploratory playtest script (not part
  of the permanent suite) driving ~185 real iterations across two careers
  with zero console errors, zero state-desync jumps, zero unhandled modals.

---

## REMAINING BACKLOG (not built yet)

Highest-value/lowest-risk items first. Each references the exact function/
file to start from — grep to confirm current line numbers before editing,
they drift as the file changes.

**Interconnections (low risk, still open)**
- **19.7 — traits/archetypes do nothing beyond a display badge.**
  `p.reputationArchetypes`/`p.traits` are tracked correctly but never read as
  a condition anywhere else. Pick ONE: (a) let "Fan Favorite" unlock
  popularity-flavored sponsors a bit earlier (`sponsorTierUnlocked()` in
  `data.js`), (b) let a relevant archetype shave a few points off a
  celebrity's `fanMin` gate (`eligibleCelebrities()` in `data.js`), or (c) let
  "Composed Veteran" reduce the odds of a punishing contract clause
  (`pickContractClause()` in `data.js`).
- **19.8 — home tier (`p.homeLevel`) is fully cosmetic.** Written correctly,
  never read except for display. Cleanest hook: build 19.1's mentor/protégé
  feature (below) and gate it on `homeLevel>=3`. If not building that in the
  same session, a smaller standalone option: a small passive
  Energy-recovery/Stress-decay bonus per tier in `advanceDay()`
  (`app_hub.js`) or `executeRace()` (`app_race.js`).
- **19.10 — cosmetic polish, zero functional risk**: dev-slots/clause
  warnings could use a subtler first-appearance animation; sponsor
  goal-progress bar has no memory of which race triggered the last goal
  (`checkSponsorObjectives()` in `app_actions.js`); Museum's Hall of Fame
  list has no delete/hide option (`loadHallOfFame()`/`appendToHallOfFame()`
  in `engine.js` support read+append only).
- **19.11 — no contextual first-time explanations** for dev slots, romance
  types, or celebrities (only documented in the Help modal). Build one
  `showFirstTimeExplainer(flagName, text)` helper in `app_actions.js`, gate
  with new `p.seen*Explainer` booleans (same pattern as `tutorialSeen`), call
  from `renderTeamScreen()`/`renderRomanceCandidatesModal()`/
  `renderCelebrityEncounterModal()`.

**Sim-touching features (medium/high risk — need a dedicated session,
read the full risk notes before starting)**
- **19.1 #1 — Tire compound choice.** Add `compound` to `s.currentSetup`,
  pick it on `renderRaceStrategyPrompt()` alongside push/attack/defend/
  saveTires, feed into `simulateRace()` (`engine.js`) as one more modifier.
- **19.1 #2 — Safety car / red flag events.** Blocked on `simulateRace()`
  having no lap loop (single score-and-sort). Cheapest scope: one extra
  `Math.random()` roll post-scoring + a compression formula + a flag threaded
  to the result modal. **Same fragile qualifying→race handoff that has
  produced a real bug before (Section 17.5 in the full history) — build a
  dedicated qualifying→race continuity regression pass alongside this.**
- **19.1 #4 — Formation-lap reaction-time mini-moment.** Reuse
  `startQuizMiniGame()`'s plumbing with one timed-tap step; scale by the
  `reaction` stat for a natural tie-in to Pass 6's stat wiring.
- **19.1 #6 — Junior/reserve driver to mentor.** New `p.protege` field,
  unlock via `homeLevel>=3`. **Highest-risk item in this backlog** — any path
  injecting a named driver onto the grid MUST go through the same
  replace-not-append discipline as `makeLegendDriver()`/
  `rebalanceGridForTeamChange()` (`engine.js`), verified the same way
  `legend_driver_test.js`/`team_switch_test.js` already verify the 22-driver
  invariant. Build the test alongside the feature, not after.
- **19.1 #7 — Mid-season team upgrade at a specific round.** Pick a round
  index at calendar-build time, call a scaled-down
  `developTeamsForNewSeason()` once mid-season when that round completes.
  Must NOT touch the player's `devSlotsUsed` (that's the separate manual-
  purchase budget) and needs the same idempotency guard pattern as
  `s.lastWrappedSeason` (`handleSeasonEnd()`).
- **19.1 #8 — Engine/gearbox grid penalties tied to reliability.** Same
  qualifying→race handoff risk as #2 above — build a dedicated continuity
  test alongside this, not as an afterthought.
- **19.1 #13 — Team switch cooldown/loyalty cost.** Add
  `p.joinedTeamOnSeason`, ramp morale for the new team's first season. Check
  against the `leadership`-driven passive morale boost in `executeRace()` —
  decide explicitly whether it still applies during cooldown (probably yes).
- **19.9 — Lap-time/best-lap layer.** Same no-lap-loop blocker as #2/#8 —
  would need to be a derived/presentational number (`plausibleLapTime()` in
  `engine.js`), not a literal simulated value. Needs a `baseLapTime` field
  per track in `data.js`.
- **19.12 — Sprint weekend isn't comprehensive.** Friday practice on a sprint
  weekend is identical to a normal weekend's; `runSprintRace()` runs
  qualifying and the race back-to-back with no separate Sprint Qualifying
  session and no strategy prompt. Three independent sub-pieces (can build any
  subset): (a) split out `runSprintQualifying()`, (b) give the Sprint its own
  2-option strategy prompt, (c) sprint-specific flavor/headlines. (a) is the
  biggest of the three but doesn't touch the main weekend's qualifying→race
  handoff.
- **19.13 — misc practice/qualifying/race polish ideas**: track-weighted quiz
  question themes, forced wet-weather quiz theme, visible Q1/Q2/Q3
  elimination framing (cosmetic only), stat-attributed post-race radio lines,
  a rare (~5-8%) late-race weather-shift event applied as a one-time score
  adjustment rather than mutating `round.weather`.

Ask if you want the full pre-compression history (prior passes 1-6) restored
for any of the above — it's still recoverable from version history, just not
duplicated here to keep this document workable.
