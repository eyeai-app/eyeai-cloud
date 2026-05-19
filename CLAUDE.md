# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

EyeAI — a mobile-first eye health screening app. The entire app is a single file: **`index.html`**. There is no build step, no bundler, no package manager. Open `index.html` directly in a browser or serve it statically.

## Development

**Run locally:**
```
python -m http.server 8080
# or
npx serve .
```
Then open `http://localhost:8080` on the device being tested. iOS testing requires HTTPS — use ngrok or a tunnel.

**There are no tests, no lint commands, and no CI pipeline.**

## Architecture

### Single-file structure
All HTML, CSS, and JS lives in `index.html` (~2000 lines). Sections are delimited by comments like `// ── SECTION NAME ───`.

### Screen system
Five screens (`<div class="screen" id="sN">`), only one active at a time via `.active` class. `go(id)` switches between them.

| ID | Purpose |
|----|---------|
| `s1` | Welcome / settings |
| `s2` | Camera / eye photo capture |
| `s3` | Photo review |
| `s4` | Vision test setup |
| `s5` | Results |

### Fullscreen overlay system
Tests run as fullscreen overlays (`.fs-ov` class, `position:fixed;inset:0;z-index:600`) layered on top of the screens. Each has `enter*()` / `exit*()` pair that toggle `.show` class.

| Overlay ID | Test | Enter/Exit functions |
|------------|------|---------------------|
| `#calOv` | Calibration slides | `enterCalib()` / `exitCalib()` |
| `#fsOv` | Letter acuity test | `enterFS()` / `exitFS()` |
| `#ctOv` | Cover test | `enterCTFS()` / `exitCTFS()` |
| `#cbOv` | Color blindness | `enterCBFS()` / `exitCBFS()` |
| `#csInstrOv` | Contrast instruction | `enterCSInstr()` / `exitCSInstr()` |
| `#csOv` | Contrast sensitivity | `enterCSFS()` / `exitCSFS()` |

z-index hierarchy: `#tapScreen` = 700, `#recoverScreen` = 690, all `.fs-ov` = 600, `#infoPopup` = 800.

### Test flow (linear)
```
Tap to Begin → Eye photos (s2→s3) → initTest() → enterCalib() [4 slides] →
calNext() → startDistCam() → runPhase('both') [acuity] →
saveTS() → runPhase('left') → saveTS() → runPhase('right') → saveTS() →
initCoverTest() → finishCoverTest() → initCBTest() → finishCBTest() →
initCSTest() → [csInstrOv] → startCSAfterInstr() → csNextPhase() loop →
showRes()
```

Every test-to-test transition uses `speakThenDo(text, rate, pauseAfterMs, callback)` — the canonical voice-then-navigate pattern. Never use `setTimeout` with a fixed guess for voice duration.

### `scores` object
```javascript
scores = { both, left, right, ct, cb, cs }
// both/left/right: SNELLEN row index (0–9) or null
// ct: { rightJump: bool, leftJump: bool } or null
// cb: { total, correct, pct } or null
// cs: { both, left, right } each a CS_LEVELS % value or null
```

### State flags
| Flag | Purpose |
|------|---------|
| `tStarted` | Blocks `updateDistBar` from auto-triggering `startVisionTest()` again |
| `tActive` | Gates `ans()` button presses during acuity test |
| `tPaused` | Set when distance goes out of range mid-test |
| `csProcessing` | Debounce guard on contrast Yes/No buttons |
| `csVoiceAsked` | Ensures contrast question is spoken only once per test session |
| `csInstrShown` | Ensures contrast instruction slide shows only once per session |

### Voice system
- `speak(key)` — speaks a translation key from the `TR` object
- `speakTxt(txt, force, rate)` — speaks a literal string
- `speakThenDo(txt, rate, pauseAfter, callback)` — speaks then waits `pauseAfter` ms before calling callback; always use this for test transitions
- All voice uses `_bestVoice` (pre-selected female English voice); iOS requires `unlockAudio()` from a user gesture before any speech

### Session recovery
`eyeai_sess` localStorage key stores `{scores, timestamp}`. On tap-to-begin, if a recent session exists, `showRecoverUI()` prompts continue or restart. `doRecover()` routes back into the correct test based on which `scores` fields are non-null — the order is: `right===null` → acuity, `ct===null` → cover test, `cb===null` → color test, `cs===null` → contrast test, else results.

### AI model
`models/consumer/` and `models/clinical/` contain TensorFlow.js model shards loaded via `loadEmbeddedModels()`. Eye photos are passed through the model in `runAIAnalysis()` → `displayAIResult()` → `updateAIRecommendation()`. The AI result appends to (never overwrites) existing acuity/contrast findings in `#actxt`.

### Key patterns to preserve
- **`resetAll()`** must exit every overlay and reset every test-specific flag/counter. When adding a new test, add its `exit*()` call and state resets here.
- **`skipTest()` / `skipTestFS()`** must also exit every overlay and null out all post-acuity scores.
- **`scores` object** is initialized in 5 places — always use `replace_all` when adding a new field.
- **`tVoiceGlobal()`** maintains a list of mute button IDs — add any new overlay's mute button ID to this list.
- **`doRecover()`** routing must mirror the test order exactly.
