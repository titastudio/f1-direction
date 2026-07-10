# FORMULA ONE DIRECTION — Project Handoff

Single-file HTML/CSS/JS Formula 1 career life sim (pixel + Tamagotchi style).

**Output:** `formula-one-direction.html`

Keep this document updated whenever code changes.

---

# BEFORE CODING

1. Work on source files, **never** the generated HTML.
2. Before changes:
   - rebuild
   - syntax check
   - run full test suite
3. Read **Architecture** first.
4. Pick a backlog item.
5. After every change:
   - rebuild
   - syntax check
   - full test suite
   - update this document.

---

# PROJECT STRUCTURE

```
shell.html        HTML template
styles.css        UI/theme
data.js           Static data (teams, tracks, sponsors, traits...)
engine.js         Core simulation & save/load
app_core.js       App singleton, creation, save pipeline
app_actions.js    Modals & interactions
app_hub.js        Home/day loop
app_race.js       Race weekends
app_screens.js    Team/Stats/Museum/etc.
app_tutorial.js   Tutorial & Help
rebuild.py        Generates index.html
```

Rebuild:

```bash
cd /home/claude/apex
python3 rebuild.py
```

If sources are lost, extract JS from the shipped HTML.

---

# TESTING

Use Puppeteer + bundled Chrome.

### Always

- rebuild
- syntax check
- run every suite

### Important

- Scroll before clicking.
- No `waitForTimeout()`.
- Use fresh element lookups after heavy computation.
- First-time explainers must open synchronously.
- Long statistical tests:
  - 1000+ trials
  - ignore sentinel values (DNFs, etc.)

---

# TEST SUITES

### Core

- `regression_test.js` — complete playthrough
- `coverage_audit.js` — responsive/layout
- `retirement_test.js` — retirement → legacy flow

### Feature suites

- `suite_economy_test.js`
- `suite_progression_test.js`
- `suite_features_test.js`
- `suite_features2_test.js`
- `suite_newfeatures_test.js`
- `suite_pass21_test.js`
- `suite_pass22_test.js`
- `suite_pass23_test.js`

Run all:

```bash
for t in regression_test coverage_audit retirement_test \
suite_economy_test suite_progression_test \
suite_features_test suite_features2_test \
suite_newfeatures_test suite_pass21_test \
suite_pass22_test suite_pass23_test
do
  timeout 120 node $t.js 2>&1 | tail -5
done
```

---

# TESTING RULES

- Every new feature gets at least one regression test.
- Reuse existing `runCheck()` pattern.
- Split very large suites instead of endlessly growing one file.
- Treat random Chrome disconnects as sandbox issues unless reproducible.
- Verify real regressions by reproducing them multiple times.

---

# KNOWN TEST PITFALLS

### Chrome instability

The sandbox has only **1 CPU core**.

Symptoms:

- Protocol error
- Detached frame
- Execution context destroyed
- Session closed

Mitigation:

- keep playthroughs short
- split heavy suites
- clean `/tmp/.org.chromium*`
- relaunch browser between large suites

---

### Statistical tests

Use enough trials.

Ignore sentinel values (DNFs) unless testing DNFs themselves.

---

### First-time explainers

Must open immediately.

Never trigger with `setTimeout()`.

---

### Cached ElementHandles

Prefer fresh lookups or

```js
page.evaluate(() => document.getElementById(id).click())
```

instead of stale handles.

---

# DOCUMENT MAINTENANCE

Whenever code changes, update:

- State Shape
- Quick Reference
- Remaining Backlog
- Changelog

Never leave this handoff document stale.

---

## DESIGN LANGUAGE

- Style: Pixel handheld/Tamagotchi aesthetic, **not** realistic racing sim.
- Layout: Fixed-width device shell (`#device`, max 430px) with centered rounded card.
- Driver display: Circular `.pod` (150px Home, 96px Race/Team) with idle animation and mood badge.
- Theme: Mint shell, cream screen, warm brown text, pastel accent colors (`.gold` = primary CTA). Fonts: **Press Start 2P** + **Nunito**.
- Reuse existing UI components before creating new ones:
  - `.pixel-panel`
  - `.pbtn` (+ variants)
  - `.chip` (+ variants)
  - `renderPixelBar()`
  - Modal components
  - `.option-grid` / `.option-card`
  - `.headline-card`
  - `.editable-name`
  - `.toast`
- Bottom navigation: 6 fixed tabs (Home, Race, Train, Team, Social, Stats).
- Help/tutorial text: short, flowing paragraphs—not `<br>` fragments.

---

## ARCHITECTURE

### App
- Everything extends the global `App` singleton via `Object.assign(App,{...})`.
- Method names **must be unique** across files.
- `App.state` is the save game.

### Team data
- `TEAMS` is **not** stored directly in `App.state`.
- Mutable team fields are synchronized through:
  - `syncTeamsToState()`
  - `syncTeamsFromState()`
- Any new mutable team field must be added to both.

### Saving
- Always call `saveState()` after changing `App.state`.
- Never call `saveGame()` directly.
- Global rules that should run after every state change belong inside `saveState()`.

### Navigation
- Bottom-nav click handlers are wired once in `App.init()`.
- New screens only update `.nav-item.active` and the status strip.

### Modals
- Every modal begins with `this.modalHeader(title)`.
- `openModal()` automatically wraps the body and wires editable names.
- If state has already advanced before the modal opens, make it **must-resolve** (no close button / overlay dismissal).

### First-time explainers
- Use `showFirstTimeExplainer()`.
- Call it **before** the real feature opens.
- Fire synchronously (never via `setTimeout`).

### Editable names
- Render with `editableName()`.
- Call `wireEditableNames()` after inserting HTML (handled automatically by `openModal()`).

### Save compatibility
- Save/Legacy/Hall-of-Fame keys include automatic migration from old key names.

### Editing workflow
- Always edit the latest file contents.
- After changes: syntax check + full regression tests.

---

## STATE SHAPE (`App.state`)

Keep this updated whenever adding/removing persistent state.

```js
{
  player:{
    // Identity
    id,name,gender,nationality,flag,age,teamId,
    attributes:{driving,mental,lifestyle},

    // Progression
    fanPopularity,mediaReputation,
    traits, reputationArchetypes,
    drivingStyle, styleProgress,

    // Career
    contract, money, homeLevel,
    careerStats,
    difficulty,

    // Progress
    trackFamiliarity,
    setupTrainingByTrack,
    trainingActionsThisWeek,
    trainingWeekTracker,
    recentQuizIds,

    // Sponsors / Journal
    sponsors,
    sponsorHistory,
    unlockedSponsorTiers,
    milestones,
    journal,
    seasonAwardsWon,

    // Academy
    protege,
    protegeRoster,

    // Team
    joinedTeamOnSeason,
    releaseClauseUsedThisContract,
    isTeamOwner,
    simRacingRep,

    // Race weekend
    mechanicalRiskMult,
    isHomeTrackBonus,
    playerTrackBests,
    _wetSetupChosen,
    _ownerAiStrategy,

    // Cosmetics
    carColorway,

    // Misc
    radioLog,
    retired,

    // First-time explainers
    seen*Explainer
  },

  // World
  grid,
  season,
  careerStartSeason,
  calendar,
  dayIndex,
  week,

  standings,
  constructorPoints,
  seasonStats,
  seasonObjectives,

  npcRoster,
  rivalId,

  newsFeed,
  history,

  facilities,
  currentSetup,
  teamMorale,

  cosmetics,
  teamState,

  trackRecords,
  trackDryStreak,

  pendingSponsorRenewals,
  guestCombativeCounts,
  rivalJournalist,

  scoutingOffer,
  pendingTierOffer,
  pendingRomanceCandidates,
  metPartner,

  lastRaceHeadline,
  seasonStartCareerWins,
  abandonedDriverNames,

  tutorialSeen,
  retiredFlag,
  careerRetirementCount
}
```

### Outside `App.state`

- `TEAMS` (`data.js`) – mutable team data (synced through `teamState`)
- Legacy (`LEGACY_KEY`) – retired player perk
- Hall of Fame (`HOF_KEY`) – retired career records

### Notes

- Use `seasonLabel()` / `seasonNumToYear()` for display.
- Team changes must sync through `syncTeamsToState()` / `syncTeamsFromState()`.
- `constructorPoints` is tracked separately from driver standings.
- `carUpgrades` is legacy only—use `teamState`/`TEAMS`.
- Update this section whenever new persistent fields are added.

---

## QUICK REFERENCE (File Map)

- Core data (teams, tracks, sponsors, names, traits, archetypes, cosmetics, romance, quiz database) → `data.js`
- Race simulation (driver strength, qualifying, race, pace, fastest lap, car setup, home track, AI progression, retirements, scouting) → `engine.js`
- Character creation, save/load, tutorials, boot flow → `app_core.js`
- Hub, navigation, day progression, constructors, controversy events → `app_hub.js`
- Activities (training, media, sponsors, mentor/academy, nightlife, explainers, sim racing) → `app_actions.js`
- Race weekend (practice, qualifying, strategy, weather, team orders, pit strategy, red flags, rivalries, multi-season callbacks, race results) → `app_race.js`
- Team/Profile/Museum/Contracts/Season wrap/Owner HQ screens → `app_screens.js`
- Tutorials & Help → `app_tutorial.js`
- Styling → `styles.css`

### Major Systems

- Skill cap → `app_core.js`
- Driving Style → `engine.js`
- Rivalries & Traits → `app_race.js`
- NPC relationships/personality → `app_actions.js`
- Mentor / Driver Academy → `app_actions.js` + `engine.js`
- Car development, facilities & contracts → `app_screens.js`
- Season progression → `app_screens.js` + `engine.js`
- Constructors' Championship → `app_hub.js` + `app_race.js`
- Team Ownership Mode → `app_core.js`, `app_screens.js`, `app_race.js`
- Legend Exhibition → `app_screens.js`
- Junior Series → `app_core.js`
- Simulator / Sim Racing → `app_race.js` + `app_actions.js`
- Driver Market → `engine.js`
- Hall of Fame / Legacy → `engine.js`

---

## EXTENSION CHECKLIST

1. New page? Prefer a modal unless it truly needs a persistent screen.
2. Every modal starts with `this.modalHeader('Title')`.
3. If a modal must be resolved before continuing, disable close button and overlay dismissal.
4. Display driver/NPC names with `editableName()` and wire using `wireEditableNames()`.
5. New screens only update `.nav-item.active` and `updateStatusStrip()`—don't add nav wiring.
6. Always modify `App.state` then call `saveState()` (never `saveGame()`).
7. New persistent state:
   - Player/save → update STATE SHAPE.
   - Team fields → also update `syncTeamsToState()` and `syncTeamsFromState()`.
8. Global rules that should run after every state change belong in `saveState()`.
9. Reuse existing UI helpers (`renderPixelBar`) and CSS before creating new ones.
10. Show first-time explainers synchronously (never via `setTimeout`).
11. Keep Help/Guide content as normal paragraphs, not `<br>` fragments.
12. Before adding a new method, grep for its name to avoid `Object.assign()` collisions.
13. After every change: rebuild → syntax check → run full test suite.
14. If you modify documented systems (STATE SHAPE, BACKLOG, etc.), update this document too.

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
- **Pass 22** – Added 17 features (weather radar, paddock vote, sponsor renewal, track records, home track, practice programs, team orders, expanded press, release clauses, simulator mode, roadmap milestones, junior series, Team Ownership Mode, microclimates, sim racing, Driver Academy, Legend Exhibition), fixed 2 real bugs (junior-series bonus clamp, Team Ownership season-wrap modal), and added 18 targeted regression tests.
- **Pass 23** – Added 22 more features (rubbering-in flavor, personal bests, shareable paddock vote, wet setup, mirror team orders, rival journalist, race preview, livery customization, Media Day, constructor rivalry, driver market news, race radio log, etc.), fixed a real tutorial timing bug that swallowed the player's first click, and added 12 targeted regression tests.
- **Pass 24** – Completed remaining Pass 23 backlog: explicit sprint Home Track bonus, constructor-rival follow-up, Driver Academy progression, real AI driver-market transfers, dedicated Owner HQ screen, fixed a multi-protégé callback bug, and added 13 targeted regression tests.
- **Pass 25** – Completed Team Ownership Mode with AI strategy calls and independent hiring/firing, added constructor-rival fallback handling, fixed 3 owner-mode bugs (teammate selection, scouting offers, contract/retirement flow), and added 6 targeted regression tests.
