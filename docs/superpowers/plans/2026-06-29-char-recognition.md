# Instant Character Recognition Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `char` game mode that plays a short random character string in Morse and scores the user's typed copy exact-match, as a fast recognition drill.

**Architecture:** A new pure-ish generator in `stationGenerator.js` returns a station-like object (with `played`/`answer` strings plus the same audio fields a calling station has) so it reuses `createMorsePlayer`. New `modeUIConfig.char`/`modeLogicConfig.char` entries, a mode radio button, and a mode-only settings accordion drive the UI. A `char` branch in `cq()`/`send()` runs the drill loop (generate → play → score → reveal → auto-advance) without any QSO exchange.

**Tech Stack:** Plain HTML/CSS/JS, Bootstrap 5.3, Webpack 5, the existing Web Audio morse engine. No new dependencies.

## Global Constraints

- No new dependencies; no webpack/build config changes.
- Mode key is exactly `char` everywhere (radio `value="char"`, `id="modeChar"`, `modeUIConfig.char`, `modeLogicConfig.char`, localStorage mode value).
- Character-set input ids exactly: `charLetters`, `charNumbers`, `charPunctuation`, `charProsigns`. Length input ids exactly: `minChars`, `maxChars`.
- Defaults: `charLetters` and `charNumbers` checked; `charPunctuation`/`charProsigns` unchecked; `minChars` = 1, `maxChars` = 5 (input range 1–10).
- Punctuation set: `.` `,` `?` `/`. Prosigns: play-form bracketed `<BK> <AR> <SK> <KN> <BT>`, answer-form plain `BK AR SK KN BT`.
- Scoring is exact match: `responseFieldText === item.answer.toUpperCase()` (responseFieldText is already trimmed + uppercased by `send()`).
- Commits must use `git commit --no-gpg-sign` (repo requires signing but no key is configured here).
- The husky pre-commit hook runs Prettier on `src/**`; write Prettier-formatted code. After committing, re-check `git status` for an unstaged Prettier reformat and, if present, commit it.
- No test runner exists (`package.json` `test` is a stub). Verification per task = `npm run build` compiles + the manual checks listed. Do not add a test framework.

## File Structure

- `src/js/stationGenerator.js` — add `getRandomCharacterString(inputs)` + the `buildCharPool` helper and character-set constants.
- `src/js/modes.js` — add `modeUIConfig.char` and `modeLogicConfig.char`.
- `src/index.html` — add the `char` mode radio button; add the "Character Recognition Settings" accordion (hidden by default); add an id to the results "Callsign" header for per-mode relabeling.
- `src/js/app.js` — extend `applyModeSettings` (settings-panel + header visibility); add `currentCharItem` state, the `getRandomCharacterString` import, `startCharDrill`/`nextCharString`, the `char` branches in `cq()`/`send()`, and clear `currentCharItem` in `resetGameState`.
- `src/js/inputs.js` — read the new fields in `getDOMInputs`; validate them in `validateInputs`.

---

### Task 1: Random character string generator

**Files:**
- Modify: `src/js/stationGenerator.js` (add constants + `buildCharPool` + `getRandomCharacterString`; place near the top-level helpers, e.g. just above `export function getCallingStation`)

**Interfaces:**
- Consumes (from the `inputs` object produced by Task 3 / `getInputs()`): `charLetters`, `charNumbers`, `charPunctuation`, `charProsigns` (booleans); `minChars`, `maxChars` (numbers); and the existing audio fields `minSpeed`, `maxSpeed`, `enableFarnsworth`, `farnsworthSpeed`, `minVolume`, `maxVolume`, `minTone`, `maxTone`, `qsb`, `qsbPercentage`.
- Produces: `getRandomCharacterString(inputs)` → object `{ played: string, answer: string, wpm, enableFarnsworth, farnsworthSpeed, volume, frequency, player: null, qsb, qsbFrequency, qsbDepth }`. `played` is the morse-engine form (prosigns bracketed, e.g. `"A5<BK>K"`); `answer` is the uppercase typed/answer form (prosigns plain, e.g. `"A5BKK"`). Task 4 reads `.played`, `.answer`, `.wpm`, `.enableFarnsworth`, `.farnsworthSpeed`, `.player`.

- [ ] **Step 1: Add the generator and helpers**

In `src/js/stationGenerator.js`, immediately before `export function getCallingStation(currentMode) {`, insert:

```javascript
// Character sets for Character Recognition mode. Each pool entry has a `play`
// form (fed to the morse engine; prosigns are bracketed so they key off the
// engine's prosign map) and an `answer` form (what the user types; prosigns are
// plain letter pairs).
const CHAR_LETTERS = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.split('');
const CHAR_NUMBERS = '0123456789'.split('');
const CHAR_PUNCTUATION = ['.', ',', '?', '/'];
const CHAR_PROSIGNS = [
  { play: '<BK>', answer: 'BK' },
  { play: '<AR>', answer: 'AR' },
  { play: '<SK>', answer: 'SK' },
  { play: '<KN>', answer: 'KN' },
  { play: '<BT>', answer: 'BT' },
];

/**
 * Builds the pool of selectable characters for Character Recognition mode from
 * the enabled character-set toggles. Returns an array of { play, answer } items.
 *
 * @param {Object} inputs - Validated inputs from getInputs().
 * @returns {Array<{play: string, answer: string}>}
 */
function buildCharPool(inputs) {
  const pool = [];
  if (inputs.charLetters) {
    CHAR_LETTERS.forEach((c) => pool.push({ play: c, answer: c }));
  }
  if (inputs.charNumbers) {
    CHAR_NUMBERS.forEach((c) => pool.push({ play: c, answer: c }));
  }
  if (inputs.charPunctuation) {
    CHAR_PUNCTUATION.forEach((c) => pool.push({ play: c, answer: c }));
  }
  if (inputs.charProsigns) {
    CHAR_PROSIGNS.forEach((p) => pool.push({ play: p.play, answer: p.answer }));
  }
  return pool;
}

/**
 * Generates a random character string for Character Recognition mode and returns
 * a station-like object playable by createMorsePlayer. The audio fields mirror
 * getCallingStation so playback uses the existing Calling Station settings.
 *
 * @param {Object} inputs - Validated inputs from getInputs(). At least one
 *   character set must be enabled (guaranteed by validateInputs).
 * @returns {Object} { played, answer, wpm, ...audio fields }
 */
export function getRandomCharacterString(inputs) {
  const pool = buildCharPool(inputs);

  const length =
    Math.floor(Math.random() * (inputs.maxChars - inputs.minChars + 1)) +
    inputs.minChars;

  let played = '';
  let answer = '';
  for (let i = 0; i < length; i++) {
    const item = pool[Math.floor(Math.random() * pool.length)];
    played += item.play;
    answer += item.answer;
  }

  return {
    played: played,
    answer: answer,
    wpm:
      Math.floor(Math.random() * (inputs.maxSpeed - inputs.minSpeed + 1)) +
      inputs.minSpeed,
    enableFarnsworth: inputs.enableFarnsworth,
    farnsworthSpeed: inputs.farnsworthSpeed || null,
    volume:
      Math.random() * (inputs.maxVolume - inputs.minVolume) + inputs.minVolume,
    frequency: Math.floor(
      Math.random() * (inputs.maxTone - inputs.minTone) + inputs.minTone
    ),
    player: null,
    qsb: inputs.qsb ? Math.random() < inputs.qsbPercentage / 100 : false,
    qsbFrequency: Math.random() * 0.45 + 0.05,
    qsbDepth: Math.random() * 0.4 + 0.6,
  };
}
```

- [ ] **Step 2: Verify the build compiles**

Run: `npm run build`
Expected: `webpack ... compiled` with only the 3 pre-existing asset-size warnings; no errors.

- [ ] **Step 3: Commit**

```bash
git add src/js/stationGenerator.js
git commit --no-gpg-sign -m "Add random character string generator for char mode"
```

---

### Task 2: Mode config, mode button, settings panel, and per-mode visibility

**Files:**
- Modify: `src/js/modes.js` (add `char` to `modeUIConfig` and `modeLogicConfig`)
- Modify: `src/index.html` (add `char` mode radio button; add the settings accordion hidden by default; add id to the Callsign `<th>`)
- Modify: `src/js/app.js` (extend `applyModeSettings`)

**Interfaces:**
- Consumes: nothing from other tasks.
- Produces: the `char` mode is selectable; `modeUIConfig.char`/`modeLogicConfig.char` exist (so `getModeConfig()`/`applyModeSettings()` never dereference undefined for `char`); DOM ids `modeChar`, `charRecognitionSettings`, `collapseCharRecognition`, `charLetters`, `charNumbers`, `charPunctuation`, `charProsigns`, `minChars`, `maxChars`, and `resultsIdentifierHeader` exist for Tasks 3 and 4. `applyModeSettings('char')` shows the settings panel and sets the identifier header to "String".

- [ ] **Step 1: Add mode config entries**

In `src/js/modes.js`, add this entry to `modeUIConfig` (after the `cwt` entry, before the closing `}` of `modeUIConfig`):

```javascript
  char: {
    showTuButton: false,
    showInfoField: false,
    infoFieldPlaceholder: '',
    showInfoField2: false,
    infoField2Placeholder: '',
    tableExtraColumn: true,
    extraColumnHeader: 'Your Answer',
    resultsHeader: 'Character Recognition Results',
  },
```

And add this entry to `modeLogicConfig` (after the `cwt` entry, before the closing `}` of `modeLogicConfig`):

```javascript
  char: {
    cqMessage: () => '',
    yourExchange: () => '',
    theirExchange: () => '',
    yourSignoff: () => '',
    theirSignoff: () => '',
    requiresInfoField: false,
    requiresInfoField2: false,
    showTuStep: false,
    modeName: 'Character Recognition',
    extraInfoFieldKey: null,
    extraInfoFieldKey2: null,
  },
```

- [ ] **Step 2: Add the mode radio button**

In `src/index.html`, inside the mode `btn-group` (`aria-label="Mode selection"`), after the SST radio+label pair (the `modeSst` input and its `>K1USN SST</label>`), insert:

```html
          <input
            type="radio"
            class="btn-check"
            name="mode"
            id="modeChar"
            value="char"
            autocomplete="off"
          />
          <label
            class="btn btn-outline-primary d-flex align-items-center justify-content-center"
            for="modeChar"
            >Char Recognition</label
          >
```

- [ ] **Step 3: Add an id to the results identifier header**

In `src/index.html`, in the `#resultsTable` `<thead>`, change the Callsign header cell from:

```html
          <th>Callsign</th>
```

to:

```html
          <th id="resultsIdentifierHeader">Callsign</th>
```

- [ ] **Step 4: Add the Character Recognition Settings accordion (hidden by default)**

In `src/index.html`, immediately after the closing `</div>` of the **Effects Settings** accordion-item (the `accordion-item` containing `#collapseEffects`) and before the closing `</div>` of the `#accordionExample` accordion, insert:

```html
      <!-- Character Recognition Settings (shown only in char mode) -->
      <div
        class="accordion-item"
        id="charRecognitionSettings"
        style="display: none"
      >
        <h2 class="accordion-header" id="headingCharRecognition">
          <button
            class="accordion-button collapsed"
            type="button"
            data-bs-toggle="collapse"
            data-bs-target="#collapseCharRecognition"
            aria-expanded="false"
            aria-controls="collapseCharRecognition"
          >
            <h5>Character Recognition Settings</h5>
          </button>
        </h2>
        <div
          id="collapseCharRecognition"
          class="accordion-collapse collapse"
          aria-labelledby="headingCharRecognition"
          data-bs-parent="#accordionExample"
        >
          <div class="accordion-body">
            <div class="container">
              <div class="row g-3 align-items-end">
                <div class="col-12">
                  <label class="form-label">Character Sets</label>
                  <div
                    class="btn-group w-100 flex-wrap"
                    role="group"
                    aria-label="Character Sets"
                  >
                    <input
                      type="checkbox"
                      class="btn-check"
                      id="charLetters"
                      autocomplete="off"
                      checked
                    />
                    <label class="btn btn-outline-primary" for="charLetters"
                      >Letters</label
                    >

                    <input
                      type="checkbox"
                      class="btn-check"
                      id="charNumbers"
                      autocomplete="off"
                      checked
                    />
                    <label class="btn btn-outline-primary" for="charNumbers"
                      >Numbers</label
                    >

                    <input
                      type="checkbox"
                      class="btn-check"
                      id="charPunctuation"
                      autocomplete="off"
                    />
                    <label class="btn btn-outline-primary" for="charPunctuation"
                      >Punctuation</label
                    >

                    <input
                      type="checkbox"
                      class="btn-check"
                      id="charProsigns"
                      autocomplete="off"
                    />
                    <label class="btn btn-outline-primary" for="charProsigns"
                      >Prosigns</label
                    >
                  </div>
                </div>

                <div class="col-12 col-md-6">
                  <label for="minChars" class="form-label"
                    >Minimum Characters</label
                  >
                  <input
                    type="number"
                    id="minChars"
                    name="minChars"
                    class="form-control"
                    value="1"
                    min="1"
                    max="10"
                    step="1"
                    required
                  />
                </div>

                <div class="col-12 col-md-6">
                  <label for="maxChars" class="form-label"
                    >Maximum Characters</label
                  >
                  <input
                    type="number"
                    id="maxChars"
                    name="maxChars"
                    class="form-control"
                    value="5"
                    min="1"
                    max="10"
                    step="1"
                    required
                  />
                </div>

                <div class="col-12">
                  <small class="text-muted"
                    >Prosigns play run-together and are typed as plain letter
                    pairs (BK, AR, SK, KN, BT).</small
                  >
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
```

- [ ] **Step 5: Extend `applyModeSettings` for the settings panel and header**

In `src/js/app.js`, in `applyModeSettings(mode)`, add the following just before the closing `}` of the function (after the extra-column-header block):

```javascript
  // Character Recognition: show its settings panel and relabel the identifier
  // column only in char mode.
  const charSettings = document.getElementById('charRecognitionSettings');
  if (charSettings) {
    charSettings.style.display = mode === 'char' ? 'block' : 'none';
  }
  const identifierHeader = document.getElementById('resultsIdentifierHeader');
  if (identifierHeader) {
    identifierHeader.textContent = mode === 'char' ? 'String' : 'Callsign';
  }
```

- [ ] **Step 6: Verify build + manual mode switching**

Run: `npm run build`
Expected: compiles with only the 3 pre-existing warnings.

Run: `npm start`, open the app.
- A "Char Recognition" button appears in the mode group.
- Selecting it reveals a "Character Recognition Settings" accordion section (Letters + Numbers pre-checked, Punctuation/Prosigns unchecked, min 1 / max 5); the results table identifier header reads "String".
- Switching back to another mode hides the section and the header reads "Callsign".
Expected: all of the above hold.

- [ ] **Step 7: Commit**

```bash
git add src/js/modes.js src/index.html src/js/app.js
git commit --no-gpg-sign -m "Add char mode config, mode button, and settings panel"
```

---

### Task 3: Read and validate Character Recognition inputs

**Files:**
- Modify: `src/js/inputs.js` (`getDOMInputs` return object + `validateInputs`)

**Interfaces:**
- Consumes (DOM ids from Task 2): `charLetters`, `charNumbers`, `charPunctuation`, `charProsigns`, `minChars`, `maxChars`; accordion id `collapseCharRecognition`.
- Produces: `getInputs()` now includes `charLetters`, `charNumbers`, `charPunctuation`, `charProsigns` (booleans) and `minChars`, `maxChars` (numbers); returns `null` when (mode is `char` and) no set is enabled, or `minChars > maxChars`, or `minChars < 1`. Task 1's generator and Task 4's drill consume these fields.

- [ ] **Step 1: Read the new fields in `getDOMInputs`**

In `src/js/inputs.js`, in the object returned by `getDOMInputs()`, add these properties just before the closing `};` (after the `cutNumbers: getSelectedCutNumbers(),` line):

```javascript

    // Character Recognition inputs
    charLetters: document.getElementById('charLetters').checked,
    charNumbers: document.getElementById('charNumbers').checked,
    charPunctuation: document.getElementById('charPunctuation').checked,
    charProsigns: document.getElementById('charProsigns').checked,
    minChars: parseInt(document.getElementById('minChars').value, 10),
    maxChars: parseInt(document.getElementById('maxChars').value, 10),
```

- [ ] **Step 2: Validate in `validateInputs`**

In `src/js/inputs.js`, in `validateInputs(inputs)`, add the following block just before `return isValid;`:

```javascript
  if (inputs.mode === 'char') {
    if (
      !inputs.charLetters &&
      !inputs.charNumbers &&
      !inputs.charPunctuation &&
      !inputs.charProsigns
    ) {
      markFieldInvalid(
        'minChars',
        'Enable at least one character set (Letters, Numbers, Punctuation, or Prosigns).'
      );
      openAccordionSection('collapseCharRecognition');
      isValid = false;
    }
    if (isNaN(inputs.minChars) || inputs.minChars < 1) {
      markFieldInvalid('minChars', 'Minimum Characters must be at least 1.');
      openAccordionSection('collapseCharRecognition');
      isValid = false;
    }
    if (inputs.minChars > inputs.maxChars) {
      markFieldInvalid(
        'minChars',
        'Minimum Characters cannot be greater than Maximum Characters!'
      );
      openAccordionSection('collapseCharRecognition');
      isValid = false;
    }
  }
```

- [ ] **Step 3: Verify build + manual validation**

Run: `npm run build`
Expected: compiles with only the 3 pre-existing warnings.

Run: `npm start`, select Char Recognition. (Full drill comes in Task 4; here verify validation wiring once Task 4's CQ branch is in — for now confirm no console errors selecting the mode and that the new inputs exist.) Open devtools console and run `document.getElementById('minChars').value` → `"1"`.
Expected: inputs present; build clean.

- [ ] **Step 4: Commit**

```bash
git add src/js/inputs.js
git commit --no-gpg-sign -m "Read and validate character recognition inputs"
```

---

### Task 4: Drill loop wiring (cq / send / next + results)

**Files:**
- Modify: `src/js/app.js` (import; `currentCharItem` state; `resetGameState`; `cq()` branch; `send()` branch; `startCharDrill`/`nextCharString`)

**Interfaces:**
- Consumes: `getRandomCharacterString(inputs)` from Task 1 (returns `{ played, answer, wpm, enableFarnsworth, farnsworthSpeed, player, ... }`); the validated input fields from Task 3; the DOM/config from Task 2 (`modeLogicConfig.char`, ids). Reuses existing `createMorsePlayer`, `audioContext`, `getInputs`, `addTableRow`, `updateAudioLock`, `updateActiveStations`, `isBackgroundStaticPlaying`, `createBackgroundStatic`.
- Produces: the full char drill — CQ starts it; Send scores exact-match, logs a row, and auto-advances; `?`/`AGN`/`AGN?` replays.

- [ ] **Step 1: Import the generator**

In `src/js/app.js`, change the import line:

```javascript
import { getYourStation, getCallingStation } from './stationGenerator.js';
```

to:

```javascript
import {
  getYourStation,
  getCallingStation,
  getRandomCharacterString,
} from './stationGenerator.js';
```

- [ ] **Step 2: Add `currentCharItem` state and clear it on reset**

In `src/js/app.js`, add a module-level state variable next to the others (after `let lastRespondingStations = null;`):

```javascript
let currentCharItem = null;
```

In `resetGameState()`, add this line (after `totalContacts = 0;`):

```javascript
  currentCharItem = null;
```

- [ ] **Step 3: Add the drill helpers**

In `src/js/app.js`, add these two functions immediately after `function nextSingleStation(responseStartTime) { ... }` (i.e. after its closing `}`):

```javascript
/**
 * Starts the Character Recognition drill: validates inputs, ensures background
 * static, disables CQ, and plays the first random string.
 */
function startCharDrill() {
  let backgroundStaticDelay = 0;
  if (!isBackgroundStaticPlaying()) {
    createBackgroundStatic();
    backgroundStaticDelay = 2;
  }
  if (getInputs() === null) return; // validateInputs marks the offending field
  document.getElementById('cqButton').disabled = true;
  nextCharString(audioContext.currentTime + backgroundStaticDelay);
}

/**
 * Generates and plays the next random character string in the drill, then focuses
 * the response field. Re-reads inputs each time so mid-drill setting changes apply
 * (and an invalid change stops the drill gracefully).
 *
 * @param {number} responseStartTime - AudioContext time to begin playback.
 */
function nextCharString(responseStartTime) {
  const responseField = document.getElementById('responseField');
  inputs = getInputs();
  if (inputs === null) {
    document.getElementById('cqButton').disabled = false;
    return;
  }

  let item = getRandomCharacterString(inputs);
  item.player = createMorsePlayer(item);
  currentCharItem = item;
  currentStationAttempts = 0;
  updateActiveStations(1);

  let endTime = item.player.playSentence(
    item.played,
    responseStartTime + Math.random() + 1
  );
  updateAudioLock(endTime);

  currentStationStartTime = endTime;
  responseField.value = '';
  responseField.focus();
}
```

- [ ] **Step 4: Branch `cq()` into the drill**

In `src/js/app.js`, in `cq()`, immediately after these lines:

```javascript
  const modeConfig = getModeConfig();
  const cqButton = document.getElementById('cqButton');
```

insert:

```javascript
  if (currentMode === 'char') {
    startCharDrill();
    return;
  }
```

- [ ] **Step 5: Branch `send()` into drill scoring**

In `src/js/app.js`, in `send()`, immediately after the empty-field guard block (the block that ends with the `}` after `if (responseFieldText === '') { ... }`) and before `console.log(\`--> Sending "${responseFieldText}"\`);`, insert:

```javascript
  if (currentMode === 'char') {
    if (currentCharItem === null) return;

    if (
      responseFieldText === '?' ||
      responseFieldText === 'AGN' ||
      responseFieldText === 'AGN?'
    ) {
      let replayTime = currentCharItem.player.playSentence(
        currentCharItem.played,
        audioContext.currentTime + 0.25
      );
      updateAudioLock(replayTime);
      currentStationAttempts++;
      return;
    }

    const correct = responseFieldText === currentCharItem.answer.toUpperCase();
    totalContacts++;
    const wpmString =
      `${currentCharItem.wpm}` +
      (currentCharItem.enableFarnsworth
        ? ` / ${currentCharItem.farnsworthSpeed}`
        : '');
    const marker = correct
      ? `<span class="text-success">&#10003; ${responseFieldText}</span>`
      : `<span class="text-danger">&#10007; ${responseFieldText}</span>`;
    addTableRow(
      'resultsTable',
      totalContacts,
      currentCharItem.answer,
      wpmString,
      currentStationAttempts,
      '',
      audioContext.currentTime - currentStationStartTime,
      marker
    );

    nextCharString(audioContext.currentTime + 1);
    return;
  }
```

- [ ] **Step 6: Verify build**

Run: `npm run build`
Expected: compiles with only the 3 pre-existing warnings; no errors.

- [ ] **Step 7: Manual drill verification**

Run: `npm start`, select **Char Recognition**.
- Press CQ: a short string (1–5 chars of letters/numbers by default) plays once.
- Type what you heard, press Send (or Enter): the result row shows the correct string under "String", your answer under "Your Answer" with a green ✓ (correct) or red ✗ (wrong), and the next string plays automatically.
- A near-miss (one wrong character) is marked ✗ (exact match, not fuzzy).
- Type `?` then Send: the current string replays without scoring.
- Enable **Prosigns**, set min=max=2, restart (Reset → CQ): when a prosign plays it sounds run-together; typing it as a plain pair (e.g. `SK`) scores correct.
- Uncheck all four sets (or set min > max) and press CQ: the drill does not start and the offending field is flagged with the Character Recognition section opened.
Expected: all of the above hold.

- [ ] **Step 8: Commit**

```bash
git add src/js/app.js
git commit --no-gpg-sign -m "Wire character recognition drill loop into cq/send"
```

---

## Self-Review

- **Spec coverage:**
  - Mode key `char`, "Char Recognition" button → Task 2.
  - Stripped-down drill loop (CQ → play once → type → score → reveal + auto-advance), `?`/`AGN` replay → Task 4.
  - Random string generation, play/answer forms, length in [minChars,maxChars], repeats, reuse of Calling Station audio fields → Task 1.
  - Four character-set toggles + min/max, defaults, "at least one set" + min≤max + min≥1 validation → Tasks 2 (HTML) & 3 (read/validate).
  - Exact-match scoring → Task 4.
  - Settings panel shown only in char mode; results table reuse with "Your Answer" extra column and per-mode "String" identifier header → Tasks 2 & 4.
  - No test runner → verification is build + manual in every task.
  - Out-of-scope items (weighting, timed sessions, bracket prosigns) → not implemented.
- **Placeholder scan:** No TBD/TODO; every code step shows complete code.
- **Type consistency:** `getRandomCharacterString(inputs)` defined in Task 1, imported/called in Task 4. Object fields `played`/`answer`/`wpm`/`enableFarnsworth`/`farnsworthSpeed`/`player` consistent between Task 1 (produced) and Task 4 (consumed). Input field names `charLetters`/`charNumbers`/`charPunctuation`/`charProsigns`/`minChars`/`maxChars` identical across Tasks 1, 2, 3. DOM ids `modeChar`, `charRecognitionSettings`, `collapseCharRecognition`, `resultsIdentifierHeader` defined in Task 2, used in Tasks 2–3. `addTableRow` call uses its real 8-arg signature (`tableName, index, callsign, wpm, attempts, stations, totalTime, extra`); `totalTime` passed as a number (`audioContext.currentTime - currentStationStartTime`) since `addTableRow` calls `.toFixed(2)` on it.
