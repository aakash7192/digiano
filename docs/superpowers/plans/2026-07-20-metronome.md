# Metronome with Adjustable BPM Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a free-running metronome to Tonik with a number-input + tap-tempo BPM control, matching the app's existing studio-hardware visual language.

**Architecture:** Everything lives in `index.html` (the app is a single dependency-free HTML file — no build step, no framework). A new `.control` block goes in the existing "Scale & sound" panel. A lookahead scheduler (`setInterval` polling an `AudioContext` clock) drives a synthesized click, independent of the existing recorder/playback transport. Two pieces of real logic (which ticks fall in the next scheduling window, and BPM implied by tap timestamps) are written as pure functions so they can be unit-tested with Node's built-in `assert`, even though the project has no test framework.

**Tech Stack:** Vanilla JS, CSS custom properties, Web Audio API. No new dependencies.

## Global Constraints

- BPM range: 30–300 (spec).
- Metronome runs independently of playback/recording state — no gating on `playing` or `recording`.
- No time signature or beat accent — uniform clicks only (spec, explicit scope decision).
- BPM persists to `localStorage` under key `tonik-bpm`, following the existing `tonik-finish` pattern (`index.html:1462`, `1466`). On/off state is NOT persisted — metronome always starts off on page load.
- Lookahead scheduler: 25ms poll interval, 100ms scheduling-ahead window (spec).
- No new colors or fonts — every value must come from the existing `:root` / `[data-theme]` custom properties (`index.html:10-115`).
- Reuse existing component styles where possible: `.control-label`, `.deck-btn` + `.playing`, `.oct-btn` press treatment, `.readout-note`-style recessed numeral chrome.
- The click's beat LED must be driven by the actual scheduled tick time, not a generic CSS `animation` loop (spec's signature detail).

---

### Task 1: Metronome control markup + styling

**Files:**
- Modify: `index.html:765-773` (HTML — new `.control` block in the Scale & sound panel)
- Modify: `index.html:256-264` (CSS — new rules, inserted before the `nudgeL`/`nudgeR` keyframes)

**Interfaces:**
- Consumes: existing CSS custom properties only (`--mono`, `--display`, `--well`, `--border`, `--text-2`, `--muted`, `--btn-hi`, `--btn-lo`, `--accent`, `--accent-deep`, `--ink`).
- Produces: DOM elements `#metro-bpm` (number input), `#metro-tap` (button), `#metro-toggle` (button, class `deck-btn metro-btn`), `#metro-led` (span inside the toggle). No JS behavior yet — later tasks wire these up.

- [ ] **Step 1: Add the HTML markup**

In `index.html`, the Octaves `.control` block currently reads (`index.html:765-773`):

```html
      <div class="control">
        <span class="control-label" id="oct-title">Octaves</span>
        <div class="oct-ctl" role="group" aria-labelledby="oct-title">
          <button class="oct-btn" id="oct-down" title="Shift all rows lower (Left Shift)">◀</button>
          <span class="oct-label" id="oct-label">3–5</span>
          <button class="oct-btn" id="oct-up" title="Shift all rows higher (Right Shift)">▶</button>
        </div>
      </div>
    </div>
```

Insert a new `.control` block immediately after the Octaves block's closing `</div>` and before the `.panel-row`'s closing `</div>`, so it reads:

```html
      <div class="control">
        <span class="control-label" id="oct-title">Octaves</span>
        <div class="oct-ctl" role="group" aria-labelledby="oct-title">
          <button class="oct-btn" id="oct-down" title="Shift all rows lower (Left Shift)">◀</button>
          <span class="oct-label" id="oct-label">3–5</span>
          <button class="oct-btn" id="oct-up" title="Shift all rows higher (Right Shift)">▶</button>
        </div>
      </div>
      <div class="control">
        <span class="control-label" id="metro-title">Metronome</span>
        <div class="metro-ctl" role="group" aria-labelledby="metro-title">
          <input type="number" class="metro-bpm" id="metro-bpm" min="30" max="300" value="120" aria-label="Beats per minute">
          <span class="metro-unit">bpm</span>
          <button class="metro-tap" id="metro-tap" type="button">Tap</button>
          <button class="deck-btn metro-btn" id="metro-toggle" type="button" aria-pressed="false"><span class="metro-led" id="metro-led" aria-hidden="true"></span>Click</button>
        </div>
      </div>
    </div>
```

- [ ] **Step 2: Add the CSS**

In `index.html`, the `.oct-label` rule is immediately followed by the nudge-animation keyframes (`index.html:256-266`):

```css
  .oct-label {
    font-family: var(--mono);
    font-size: 13px;
    color: var(--text-2);
    min-width: 38px;
    text-align: center;
    font-variant-numeric: tabular-nums;
  }

  @keyframes nudgeL { from { transform: translateX(-14px); } to { transform: none; } }
```

Insert the metronome rules between them:

```css
  .oct-label {
    font-family: var(--mono);
    font-size: 13px;
    color: var(--text-2);
    min-width: 38px;
    text-align: center;
    font-variant-numeric: tabular-nums;
  }

  /* ---------- Metronome ---------- */
  .metro-ctl { display: flex; align-items: center; gap: 6px; }
  .metro-bpm {
    font-family: var(--mono);
    font-size: 15px;
    font-variant-numeric: tabular-nums;
    width: 52px;
    text-align: center;
    padding: 7px 4px;
    border-radius: 4px;
    background: var(--well);
    border: 1px solid var(--border);
    color: var(--text-2);
  }
  .metro-bpm::-webkit-inner-spin-button,
  .metro-bpm::-webkit-outer-spin-button { -webkit-appearance: none; margin: 0; }
  .metro-bpm { -moz-appearance: textfield; }
  .metro-unit {
    font-family: var(--display);
    font-size: 10px;
    letter-spacing: .2em;
    text-transform: uppercase;
    color: var(--muted);
  }
  .metro-tap {
    font-size: 13px;
    padding: 9px 12px;
    border-radius: 5px;
    border: 1px solid var(--border);
    background: linear-gradient(180deg, var(--btn-hi), var(--btn-lo));
    color: var(--muted);
    cursor: pointer;
    transition: color .15s;
  }
  .metro-tap:hover { color: var(--text); }
  .metro-tap.tapped {
    color: var(--ink);
    background: linear-gradient(180deg, var(--accent), var(--accent-deep));
    border-color: var(--accent-deep);
  }
  .metro-led {
    display: inline-block;
    width: 7px; height: 7px;
    border-radius: 50%;
    background: currentColor;
    opacity: .35;
    margin-right: 7px;
    vertical-align: middle;
    transition: opacity .06s ease-out, transform .06s ease-out;
  }
  .metro-led.lit { opacity: 1; transform: scale(1.4); }

  @keyframes nudgeL { from { transform: translateX(-14px); } to { transform: none; } }
```

Note: `.metro-led` uses `background: currentColor` deliberately — it inherits `--text-2` when the toggle is inactive and `--ink` when `.metro-btn.playing` sets `color: var(--ink)` (from the existing `.deck-btn.playing` rule at `index.html:519-523`), so no extra per-theme rule is needed.

- [ ] **Step 3: Verify the markup and styles are well-formed**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const ids = ['metro-bpm', 'metro-tap', 'metro-toggle', 'metro-led', 'metro-title'];
ids.forEach(id => {
  const matches = html.match(new RegExp('id=\"' + id + '\"', 'g')) || [];
  if (matches.length !== 1) throw new Error(id + ': expected exactly 1 occurrence, found ' + matches.length);
});
console.log('OK: all metronome element ids present exactly once');
"
```

Expected: `OK: all metronome element ids present exactly once`

- [ ] **Step 4: Manual visual check**

```bash
open index.html
```

In the browser: confirm the new "Metronome" control appears in the Scale & sound panel after Octaves, with a `120` numeral, `bpm` label, `Tap` button, and `Click` button (inert — clicking does nothing yet, that's expected). Click each of the four finish swatches (Walnut/Ebony/Ivory/Indigo) and confirm the new control re-themes along with everything else (no hardcoded colors). Narrow the browser window and confirm the control wraps onto its own line without overlapping other controls.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add metronome control markup and styling

Static UI only — BPM input, tap button, and click toggle rendered in
the Scale & sound panel, reusing existing button/readout chrome.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Metronome state and pure scheduling/tap-tempo logic

**Files:**
- Modify: `index.html:1467` (JS — new `/* ---------- Metronome ---------- */` section, inserted after the Finishes block and before the Computer keyboard block)

**Interfaces:**
- Consumes: nothing new (DOM elements from Task 1 are grabbed here for later tasks to use).
- Produces:
  - DOM refs: `metroBpmEl`, `metroTapEl`, `metroToggleEl`, `metroLedEl`
  - Constants: `METRO_LOOKAHEAD` (0.1, seconds), `METRO_INTERVAL` (25, ms)
  - State: `let bpm = 120`, `let metronomeOn = false`, `let metroTimer = null`, `let metroNextTick = 0`
  - Pure functions: `ticksInWindow(nextTick, horizonEnd, bpmValue) -> { ticks: number[], next: number }`, `bpmFromTaps(times: number[]) -> number | null`

- [ ] **Step 1: Write the unit test for the pure functions**

These functions don't exist yet, so this test is expected to fail. Save this as a reusable check — you'll re-run the exact same command in Step 4:

```bash
node -e "
const fs = require('fs');
const assert = require('assert');
const src = fs.readFileSync('index.html', 'utf8');
const grab = name => {
  const start = src.indexOf('function ' + name);
  if (start === -1) throw new Error(name + ' not found in index.html');
  const braceStart = src.indexOf('{', start);
  let depth = 0, i = braceStart;
  for (; i < src.length; i++) {
    if (src[i] === '{') depth++;
    if (src[i] === '}') { depth--; if (depth === 0) break; }
  }
  return src.slice(start, i + 1);
};
eval(grab('ticksInWindow') + '\n' + grab('bpmFromTaps'));
assert.deepStrictEqual(ticksInWindow(0, 1, 120).ticks, [0, 0.5]);
assert.strictEqual(ticksInWindow(0, 1, 120).next, 1);
assert.strictEqual(ticksInWindow(0, 0.4, 120).ticks.length, 1);
assert.strictEqual(bpmFromTaps([0, 500, 1000, 1500]), 120);
assert.strictEqual(bpmFromTaps([0]), null);
console.log('OK');
"
```

- [ ] **Step 2: Run it to verify it fails**

Run the command from Step 1.
Expected: `Error: ticksInWindow not found in index.html`

- [ ] **Step 3: Implement the Metronome section**

In `index.html`, the Finishes block ends and the Computer keyboard block begins like this (`index.html:1465-1469`):

```js
  swatches.forEach(s => s.addEventListener("click", () => setFinish(s.dataset.finish)));
  let savedFinish = "walnut";
  try { savedFinish = localStorage.getItem("tonik-finish") || "walnut"; } catch (e) {}
  setFinish(savedFinish);

  /* ---------- Computer keyboard ---------- */
```

Insert the new section between `setFinish(savedFinish);` and the Computer keyboard comment:

```js
  swatches.forEach(s => s.addEventListener("click", () => setFinish(s.dataset.finish)));
  let savedFinish = "walnut";
  try { savedFinish = localStorage.getItem("tonik-finish") || "walnut"; } catch (e) {}
  setFinish(savedFinish);

  /* ---------- Metronome ---------- */
  const metroBpmEl = document.getElementById("metro-bpm");
  const metroTapEl = document.getElementById("metro-tap");
  const metroToggleEl = document.getElementById("metro-toggle");
  const metroLedEl = document.getElementById("metro-led");

  const METRO_LOOKAHEAD = 0.1;   // seconds scheduled ahead of ac.currentTime
  const METRO_INTERVAL = 25;     // ms between scheduler polls
  let bpm = 120;
  let metronomeOn = false;
  let metroTimer = null;
  let metroNextTick = 0;

  // Pure: which tick times (seconds, ac.currentTime domain) fall before
  // horizonEnd starting from nextTick, and where the next tick after them
  // lands. No DOM/Audio access, so it's unit-testable in isolation.
  function ticksInWindow(nextTick, horizonEnd, bpmValue) {
    const ticks = [];
    let t = nextTick;
    const step = 60 / bpmValue;
    while (t < horizonEnd) { ticks.push(t); t += step; }
    return { ticks, next: t };
  }

  // Pure: BPM implied by a list of tap timestamps (ms), from the average
  // of the intervals between them. Needs at least 2 taps.
  function bpmFromTaps(times) {
    if (times.length < 2) return null;
    const intervals = [];
    for (let i = 1; i < times.length; i++) intervals.push(times[i] - times[i - 1]);
    const avgMs = intervals.reduce((a, b) => a + b, 0) / intervals.length;
    return 60000 / avgMs;
  }

  /* ---------- Computer keyboard ---------- */
```

- [ ] **Step 4: Run the test again to verify it passes**

Run the exact command from Step 1.
Expected: `OK`

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add metronome state and pure scheduling/tap-tempo helpers

ticksInWindow and bpmFromTaps are pure functions covered by Node
assert-based unit tests, extracted from the embedded script since
the project has no test framework.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Click synthesis, lookahead scheduler, and toggle wiring

**Files:**
- Modify: `index.html` (JS — insert into the Metronome section created in Task 2, after the `bpmFromTaps` function and before the `/* ---------- Computer keyboard ---------- */` comment)

**Interfaces:**
- Consumes: `audio()` (`index.html:880-884`), `ticksInWindow`, `bpm`, `metronomeOn`, `metroTimer`, `metroNextTick`, `METRO_LOOKAHEAD`, `METRO_INTERVAL`, `metroToggleEl`, `metroLedEl` (all from Task 2)
- Produces: `playClick(ac, time)`, `flashLed()`, `metroScheduler()`, `startMetronome()`, `stopMetronome()`; wires a click listener on `metroToggleEl`

- [ ] **Step 1: Implement click synthesis, scheduler, and toggle**

Insert immediately after the `bpmFromTaps` function from Task 2 (still before `/* ---------- Computer keyboard ---------- */`):

```js
  function playClick(ac, time) {
    const osc = ac.createOscillator();
    const gain = ac.createGain();
    osc.type = "sine";
    osc.frequency.value = 1000;
    gain.gain.setValueAtTime(0.0001, time);
    gain.gain.exponentialRampToValueAtTime(0.35, time + 0.001);
    gain.gain.exponentialRampToValueAtTime(0.0001, time + 0.04);
    osc.connect(gain).connect(ac.destination);
    osc.start(time);
    osc.stop(time + 0.05);
    setTimeout(flashLed, Math.max(0, (time - ac.currentTime) * 1000));
  }

  function flashLed() {
    metroLedEl.classList.add("lit");
    setTimeout(() => metroLedEl.classList.remove("lit"), 80);
  }

  function metroScheduler() {
    const ac = audio();
    const { ticks, next } = ticksInWindow(metroNextTick, ac.currentTime + METRO_LOOKAHEAD, bpm);
    ticks.forEach(t => playClick(ac, t));
    metroNextTick = next;
  }

  function startMetronome() {
    if (metronomeOn) return;
    metronomeOn = true;
    metroNextTick = audio().currentTime + 0.05;
    metroScheduler();
    metroTimer = setInterval(metroScheduler, METRO_INTERVAL);
    metroToggleEl.classList.add("playing");
    metroToggleEl.setAttribute("aria-pressed", "true");
  }

  function stopMetronome() {
    if (!metronomeOn) return;
    metronomeOn = false;
    clearInterval(metroTimer);
    metroTimer = null;
    metroToggleEl.classList.remove("playing");
    metroToggleEl.setAttribute("aria-pressed", "false");
  }

  metroToggleEl.addEventListener("click", () => {
    if (metronomeOn) stopMetronome(); else startMetronome();
  });
```

- [ ] **Step 2: Verify the embedded script is syntactically valid**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const script = html.match(/<script>([\s\S]*)<\/script>/)[1];
fs.writeFileSync('/tmp/tonik-script-check.js', script);
"
node --check /tmp/tonik-script-check.js
```

Expected: no output, exit code 0.

- [ ] **Step 3: Manual behavior check**

```bash
open index.html
```

In the browser: click the **Click** button. Confirm the button switches to the accent-colored active look (matching Play/Stop's active state), a click sound plays roughly twice per second (120 bpm default), and the small dot in the button visibly pulses in time with each click. Click **Click** again and confirm it stops immediately and the button returns to its inactive look. Open the browser console (right-click → Inspect → Console) and confirm there are no errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Wire up metronome click synthesis and lookahead scheduler

Toggle button starts/stops a self-scheduled click independent of
the recorder/transport; the beat LED is flipped by the scheduler
itself so it's always in sync with the actual audio timing.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: BPM input wiring and persistence

**Files:**
- Modify: `index.html` (JS — insert into the Metronome section, after the toggle wiring from Task 3)

**Interfaces:**
- Consumes: `metroBpmEl` (Task 2), `bpm` (Task 2, now reassigned here)
- Produces: `setBpm(value)` — clamps to 30-300, updates `bpm`, reflects into `metroBpmEl.value`, persists to `localStorage["tonik-bpm"]`. Later tasks (Task 5) call this instead of assigning `bpm` directly.

- [ ] **Step 1: Implement `setBpm`, the input listener, and startup restore**

Insert immediately after the `metroToggleEl.addEventListener(...)` block from Task 3:

```js
  function setBpm(value) {
    bpm = Math.min(300, Math.max(30, Math.round(value)));
    metroBpmEl.value = bpm;
    try { localStorage.setItem("tonik-bpm", String(bpm)); } catch (e) {}
  }

  metroBpmEl.addEventListener("change", () => {
    setBpm(parseInt(metroBpmEl.value, 10) || 120);
  });

  let savedBpm = 120;
  try { savedBpm = parseInt(localStorage.getItem("tonik-bpm"), 10) || 120; } catch (e) {}
  setBpm(savedBpm);
```

- [ ] **Step 2: Verify the embedded script is syntactically valid**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const script = html.match(/<script>([\s\S]*)<\/script>/)[1];
fs.writeFileSync('/tmp/tonik-script-check.js', script);
"
node --check /tmp/tonik-script-check.js
```

Expected: no output, exit code 0.

- [ ] **Step 3: Manual behavior check**

```bash
open index.html
```

In the browser: change the BPM field to `90` and click elsewhere to commit it. Open the console and run `localStorage.getItem('tonik-bpm')` — confirm it returns `"90"`. Reload the page and confirm the BPM field still shows `90`. Click **Click** and confirm the tick rate is audibly slower than the earlier 120 bpm test (~1.5 clicks/sec vs ~2/sec). Try typing `999` into the field and committing it — confirm it clamps to `300`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Wire up BPM input with clamping and localStorage persistence

BPM survives reloads via tonik-bpm, following the same pattern as
the existing tonik-finish theme key. On/off state is intentionally
not persisted.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Tap tempo

**Files:**
- Modify: `index.html` (JS — insert into the Metronome section, after the BPM wiring from Task 4, as the last thing before `/* ---------- Computer keyboard ---------- */`)

**Interfaces:**
- Consumes: `metroTapEl` (Task 2), `bpmFromTaps` (Task 2), `setBpm` (Task 4)
- Produces: `tapTimes` (module-level array), a click listener on `metroTapEl`

- [ ] **Step 1: Implement tap tempo**

Insert immediately after the `setBpm(savedBpm);` line from Task 4, still before `/* ---------- Computer keyboard ---------- */`:

```js
  let tapTimes = [];
  metroTapEl.addEventListener("click", () => {
    const now = performance.now();
    if (tapTimes.length && now - tapTimes[tapTimes.length - 1] > 2000) tapTimes = [];
    tapTimes.push(now);
    if (tapTimes.length > 5) tapTimes.shift();
    const computed = bpmFromTaps(tapTimes);
    if (computed !== null) setBpm(computed);
    metroTapEl.classList.add("tapped");
    setTimeout(() => metroTapEl.classList.remove("tapped"), 120);
  });
```

- [ ] **Step 2: Verify the embedded script is syntactically valid**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const script = html.match(/<script>([\s\S]*)<\/script>/)[1];
fs.writeFileSync('/tmp/tonik-script-check.js', script);
"
node --check /tmp/tonik-script-check.js
```

Expected: no output, exit code 0.

- [ ] **Step 3: Manual behavior check (full feature smoke test)**

```bash
open index.html
```

In the browser:
1. Click **Tap** four times at a steady pace, about half a second apart (tap along to "1... 2... 3... 4..."). Confirm the BPM field updates to something close to `120` (exact value depends on your timing) and the Tap button briefly flashes its active color on each press.
2. Wait 3 seconds without tapping, then tap twice quickly (about 300ms apart). Confirm the BPM jumps to a new value near `200` rather than being dragged toward the earlier ~120 average — this confirms the 2-second tap-history reset works.
3. With the metronome **Click** running, tap a new tempo and confirm the click rate audibly changes without needing to toggle Click off and back on.
4. Switch between all four finishes once more and confirm the whole control (BPM field, Tap, Click, LED) still looks correct in each.
5. Resize the window to a narrow/mobile width and confirm the Metronome control still lays out cleanly and the Scale & sound panel's collapse toggle still works.
6. Check the console one more time for errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Add tap tempo to the metronome

Tap intervals average into a BPM via bpmFromTaps; a >2s gap between
taps resets the history so stale timing doesn't skew a fresh tempo.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```
