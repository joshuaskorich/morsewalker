# Instant Character Recognition Mode — Design Spec

**Date:** 2026-06-29
**Branch:** `feature/char-recognition` (to be created)

## Goal

Add a new game mode, **Character Recognition**, that plays a short random
character string in Morse and asks the user to type what they heard — a fast,
stripped-down copying drill. It reuses the existing audio engine, mode system,
settings UI, and results table.

## Mode key

The mode value/key is `char` everywhere it appears (radio `value="char"`,
`id="modeChar"`, `modeUIConfig.char`, `modeLogicConfig.char`,
`localStorage` mode value). Button label: **"Char Recognition"**.

## Interaction (the drill loop)

A stripped-down drill — no CQ exchange, 5NN, or signoff:

1. User presses **CQ** (the existing button; no relabel needed).
2. A random string is generated and played **once**.
3. User types what they heard into the existing `responseField` and presses
   **Send**.
4. The answer is scored **exact-match**. The result is logged, the correct
   string is revealed, and the drill **auto-advances** to a newly generated
   string (reveal + advance on both correct and wrong answers).
5. Typing `?`, `AGN`, or `AGN?` replays the current string without scoring
   (reuse existing repeat handling).

There is no "must get it right to advance" gate; every submission scores and
advances.

## Random string generation

New function in `src/js/stationGenerator.js`:

```
getRandomCharacterString(inputs) -> {
  played: string,   // morse-engine form, prosigns bracketed, e.g. "A5<BK>K"
  answer: string,   // typed/answer form, prosigns as letters, e.g. "A5BKK"
  ...audio fields    // same shape a calling station needs for createMorsePlayer
}
```

- **Character pool** is assembled from the four enabled toggles:
  - Letters: `A`–`Z`
  - Numbers: `0`–`9`
  - Punctuation: `.` `,` `?` `/`
  - Prosigns: `BK`, `AR`, `SK`, `KN`, `BT`
- Each pool entry carries two forms: a **play form** and an **answer form**.
  For letters/numbers/punctuation both forms are identical. For prosigns the
  play form is the bracketed token the morse engine understands (`<bk>`,
  `<ar>`, `<sk>`, `<kn>`, `<bt>`) and the answer form is the plain letter pair
  (`BK`, `AR`, `SK`, `KN`, `BT`).
- **Length** is a random integer in `[inputs.minChars, inputs.maxChars]`
  (inclusive). Each position independently picks a random pool entry; repeats
  are allowed.
- `played` = concatenation of play forms; `answer` = concatenation of answer
  forms. Both are uppercase.
- The returned object includes the same audio fields a calling station has
  (wpm, tone, volume, qsb, qrn, farnsworth, etc.), drawn from the existing
  **Calling Station** settings exactly as `getCallingStation` does, so the
  string is played by `createMorsePlayer()` with the same realism controls.
  (Set min speed = max speed for a constant drill speed.)

## Scoring

Exact match only: `typed.trim().toUpperCase() === answer.toUpperCase()`.
The fuzzy `compareStrings` is **not** used in this mode. Because scoring
compares the full concatenated answer string, there is no prosign/letter
segmentation ambiguity (e.g. a generated `B`+`K` and a generated prosign `BK`
both expect typed `BK`, which is correct either way).

## Settings UI

A new accordion section, **"Character Recognition Settings"**, added to the
settings accordion stack in `src/index.html`:

- **Character set toggles** — a `btn-check` checkbox group (same pattern as the
  callsign-format group):
  - `charLetters` (checked by default)
  - `charNumbers` (checked by default)
  - `charPunctuation` (unchecked)
  - `charProsigns` (unchecked)
- **Length** — two numeric inputs (same pattern as min/max stations):
  - `minChars` — value 1, min 1, max 10, step 1
  - `maxChars` — value 5, min 1, max 10, step 1

**Visibility:** this section is shown only when the current mode is `char`, and
hidden otherwise. `applyModeSettings` (or an equivalent hook called from
`changeMode`/init) toggles the section's visibility by mode. Existing sections
(Your Station, Calling Stations) remain visible because the drill reuses the
Calling Station audio settings. The Pile-Up min/max **station-count** settings
do not apply in `char` mode and are ignored (left visible; harmless).

## Reading & validating settings

`getInputs()` (`src/js/inputs.js`) reads the new fields into the inputs object:

```
charLetters: boolean,
charNumbers: boolean,
charPunctuation: boolean,
charProsigns: boolean,
minChars: number,
maxChars: number,
```

Validation (applied when mode is `char`):
- At least one of the four character-set toggles must be enabled.
- `minChars >= 1` and `minChars <= maxChars`.

If invalid, `getInputs()` returns `null` (consistent with existing validation),
which blocks the drill from starting; the offending control is flagged using
the existing invalid-state mechanism (`clearAllInvalidStates` / the pattern
used for other invalid inputs), and the user is shown why.

## Mode config

- `modeUIConfig.char`: `showTuButton: false`, `showInfoField: false`,
  `showInfoField2: false`, `tableExtraColumn: true`,
  `extraColumnHeader: 'Your Answer'`, `resultsHeader: 'Character Recognition
  Results'`.
- `modeLogicConfig.char`: minimal entry with `showTuStep: false`,
  `modeName: 'Character Recognition'`, and no-op/empty exchange functions
  (the drill branch never calls them, but the entry must exist so
  `getModeConfig`/`applyModeSettings` don't dereference undefined).

## Drill flow in app.js

- Add a new mode radio button to the mode `btn-group` in `src/index.html`
  (same markup as the others), `id="modeChar"`, `value="char"`,
  label "Char Recognition".
- `cq()`: when `currentMode === 'char'`, branch to a drill start that
  generates a string via `getRandomCharacterString(getInputs())`, builds a
  player via `createMorsePlayer(...)`, plays `played`, focuses `responseField`,
  and stores the current `answer`/string object as the active item. No CQ
  message is played.
- `send()`: when `currentMode === 'char'`, branch to drill scoring:
  - `?`/`AGN`/`AGN?` → replay `played`, no scoring.
  - Otherwise score exact-match against `answer`, log a results row, reveal the
    correct string, increment the contact counter, then generate + play the
    next string.
- **Results row:** reuse the existing results table. The identifier column
  shows the correct `answer` string; the mode-specific extra column ("Your
  Answer") shows what the user typed, visually marked correct/incorrect
  (e.g. ✓ / ✗). WPM and elapsed time reuse the existing computations. (Exact
  `addTableRow` argument wiring is finalized during planning against the real
  signature.)

## Testing

This project has **no test runner** — `package.json`'s `test` script is a
placeholder stub and there is no jest/mocha/vitest dependency. Adding a test
stack is out of scope for this feature. Verification is therefore:

1. `npm run build` compiles without errors.
2. Manual drill verification via `npm start`: each enabled set appears, lengths
   stay within min/max, prosigns sound run-together but are typed as plain
   pairs and score correctly, exact-match marks near-misses wrong, `?` replays,
   the loop auto-advances logging correct/incorrect, and disabling all four
   sets (or min > max) blocks the drill with an invalid-state notice.

The generator and matching are pure functions; the plan keeps them isolated and
small so this manual verification is tractable.

## Out of scope

- Adjustable per-character weighting / Koch-method ordering.
- Timed sessions, streaks, or accuracy statistics beyond the results table.
- Configurable symbol sets beyond the four toggles above.
- Bracketed prosign typing (explicitly rejected in favor of plain letter pairs).
