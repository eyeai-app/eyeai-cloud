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
All HTML, CSS, and JS lives in `index.html` (~2170 lines). Sections are delimited by comments like `// ── SECTION NAME ───`.

### Screen system
Seven screens (`<div class="screen" id="...">`), only one active at a time via `.active` class. `go(id)` switches between them.

| ID | Purpose |
|----|---------|
| `sWelcome` | Welcome logo screen — shown immediately after tap, welcome voice plays here |
| `sWho` | "Who will be doing the screening today?" — Myself / My child / Clinical selection |
| `s1` | Settings — voice toggle, terms checkbox, AI model status, Start scan button |
| `s2` | Camera / eye photo capture |
| `s3` | Photo review |
| `s4` | Vision test setup + distance check |
| `s5` | Results |

### Pre-test flow (tap → camera)
```
Tap to Begin
  → onTapBegin(): tap screen hides immediately → go('sWelcome')
  → welcome voice plays on sWelcome (via speakThenDo + _voiceReadyPromise)
  → goWho() → go('sWho') + whoEnter()
  → 500ms → "Who will be doing the screening today?" voice
  → user selects Myself or My child
  → whoSelect() → speakThenDo(..., 1500ms, proceedToS1)
  → proceedToS1(): loadSession() check → go('s1') [or showRecoverUI()]
  → user checks terms → Start scan → go('s2') + initCam()
```

`childMode` (global boolean) is set by `whoSelect('child')` / `whoAgeConfirm()` and cleared in `resetAll()`. It flags child-mode for future test adaptations. The `whoShowCards()` function resets the sWho panel to its default card view.

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
Eye photos (s2→s3) → initTest() → enterCalib() [4 slides] →
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
// ct: { right: 'still'|'moved'|'jumped', left: 'still'|'moved'|'jumped' } or null
// cb: { total, correct, pct } or null
// cs: { both, left, right } each a CS_LEVELS % value or null
```

### State flags
| Flag | Purpose |
|------|---------|
| `childMode` | Set by sWho screen when scanning a child aged 3+; cleared in `resetAll()` |
| `tStarted` | Blocks `updateDistBar` from auto-triggering `startVisionTest()` again |
| `tActive` | Gates `ans()` button presses during acuity test |
| `tPaused` | Set when distance goes out of range mid-test |
| `csProcessing` | Debounce guard on contrast Yes/No buttons |
| `csVoiceAsked` | Ensures contrast question is spoken only once per test session |
| `csInstrShown` | Ensures contrast instruction slide shows only once per session |

### Eye capture — perfect-position system (`startQ()`)
The 250ms quality loop in `startQ()` runs on s2. When MediaPipe face mesh confirms the eye is centered:
- Oval turns **green** (`oval good`) immediately on the first good tick
- Voice says **"Perfect, hold still."** once (debounced 3s via `lastVoice.perfect`)
- Status shows **"Hold still…"** for 27 ticks, then **"HOLD 5/4/3/2/1"** for the last 5
- Spoken countdown: 3, 2, 1 (at ticks HOLD_NEEDED−3/−2/−1)
- If eye drifts out during countdown: `wasInCountdown` (derived from `prevGoodTick`) triggers **"Almost there, keep steady."** instead of the generic centering message
- `HOLD_NEEDED=32` ticks × 250ms = 8s hold required before auto-capture

Distance measurement uses only the `near` mode (40cm target, eye-span range 0.12–0.23). The intermediate distance mode was removed.

### Voice system
- `speak(key)` — speaks a translation key from the `TR` object
- `speakTxt(txt, force, rate)` — speaks a literal string; `force=true` cancels any ongoing speech first
- `speakThenDo(txt, rate, pauseAfter, callback)` — speaks then waits `pauseAfter` ms before calling callback; handles `onerror` as well as `onend`; always use this for screen transitions
- `_voiceReadyPromise` — resolved by `initVoices()` on page load; defer any speech that must use the best voice by chaining off this promise
- All voice uses `_bestVoice` (pre-selected female English voice); iOS requires `unlockAudio()` from a user gesture before any speech

### Session recovery
`eyeai_sess` localStorage key stores `{scores, timestamp}`. `proceedToS1()` checks for a saved session after the sWho selection. `showRecoverUI()` displays the recover overlay; `doRecover()` routes back into the correct test based on which `scores` fields are non-null — the order is: `right===null` → acuity, `ct===null` → cover test, `cb===null` → color test, `cs===null` → contrast test, else results.

### AI model
TensorFlow.js models are fetched from a Cloudflare Worker (`eyeai-cloud.berlin-davidfischer.workers.dev`) and loaded via `loadEmbeddedModels()` on page load. Eye photos are passed through the model in `runAIAnalysis()` → `displayAIResult()` → `updateAIRecommendation()`. The AI result appends to (never overwrites) existing acuity/contrast findings in `#actxt`.

### Key patterns to preserve
- **`resetAll()`** must exit every overlay and reset every test-specific flag/counter, including `childMode=false;whoShowCards()`. When adding a new test, add its `exit*()` call and state resets here.
- **`skipTest()` / `skipTestFS()`** must also exit every overlay and null out all post-acuity scores.
- **`scores` object** is initialized in 5 places — always use `replace_all` when adding a new field.
- **`tVoiceGlobal()`** maintains a list of mute button IDs — add any new overlay's mute button ID to this list.
- **`doRecover()`** routing must mirror the test order exactly.
- **`speakThenDo` not `setTimeout`** — never guess voice duration with a fixed timeout for any transition.
