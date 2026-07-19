# Metronome with adjustable BPM

## Goal

Add a metronome to Tonik: a click track the player can turn on/off at any
time, with a BPM they can set numerically or by tapping.

## Placement

A new `.control` block inside the existing "Scale & sound" panel
(`index.html:743-777`), placed after the Octaves control. This panel sits
above the Studio tabs (Perform/Mix), so the control is visible regardless of
which Studio view is active — a single global control, no duplication.

## UI

Tonik's visual language is skeuomorphic studio hardware — bezeled gradient
buttons, mono numeric readouts, wide-tracked uppercase display-font labels,
accent glow on active state, all driven by per-finish CSS custom properties.
The metronome control reuses that vocabulary directly rather than
introducing anything new, so it re-themes automatically across
Walnut/Ebony/Ivory/Indigo.

New `.control` block in the panel row, after Octaves:

```
Metronome
┌────────────┐  ┌──────┐  ┌───────────┐
│  120  bpm  │  │ Tap  │  │ ● Click   │
└────────────┘  └──────┘  └───────────┘
  mono numeral    tap       toggle (.deck-btn),
  input, reuses    tempo    dot = beat LED
  .readout-note
  chrome
```

- **BPM input** — `<input type="number" min="30" max="300" value="120">`
  styled with the recessed `.readout-note` chrome (`--well` background,
  `--border`, `--mono` font, tabular numerals) instead of a native spinner
  look; native up/down arrows hidden via CSS, arrow keys still work.
- **Tap button** — small `.oct-btn`-style square button labeled "Tap";
  press feedback matches the existing `.root-btn` pressed-state treatment.
- **Toggle button** — `.deck-btn` labeled "● Click", using the existing
  `.playing` active-state class (same one `#mix-play` uses) when running.

**Signature detail — audio-synced beat LED:** the `●` inside the toggle
button is not a static glyph or a generic CSS `animation`-driven blink. It
is flipped to `--accent` and pulsed by JS exactly when the scheduler fires
each click (see Scheduling below), so the light is only ever telling the
truth about the actual audio timing — the same reason a real metronome's
tempo light is trustworthy and a cosmetic approximation wouldn't be. This
is the one deliberate craft detail; everything else in the control reuses
existing, unmodified component styles.

## Behavior

The metronome is independent of the recorder/transport. It does not start or
stop based on playback or recording state — it's a free-running click loop
the player toggles manually, usable while practicing, while playing live, or
as a click track while recording a take.

### BPM input

Typing a new BPM value updates the running tempo immediately (takes effect
on the next scheduled tick — no need to restart the click loop).

### Tap tempo

Each click of "Tap" records a timestamp (`performance.now()`). BPM is
computed from the rolling average of the intervals between the last few taps
(e.g. last 4). If more than 2 seconds pass between taps, the tap history
resets (avoids a stale first tap skewing the average after a long pause).
Computed BPM updates the number input and, if the metronome is running,
takes effect on the next tick like a manual BPM edit.

### Scheduling

`setInterval` timers drift under load and are not sample-accurate, so the
metronome uses a standard lookahead scheduler:

- A `setInterval` runs every 25ms (the "scheduler tick").
- It maintains `nextTickTime`, an absolute time in the `AudioContext`'s
  clock (`ac.currentTime` domain).
- On each scheduler tick, while `nextTickTime` falls within the next 100ms
  (the lookahead window), a click is scheduled precisely at `nextTickTime`
  via the Web Audio API, and `nextTickTime` advances by `60 / bpm` seconds.
- Turning the metronome off clears the `setInterval` and stops scheduling
  further clicks; already-scheduled clicks (at most ~100ms out) are allowed
  to finish naturally.

### Click sound

No existing click/tick sound exists in the app. A dedicated `playClick(ac,
time)` function synthesizes one using the same oscillator + gain-envelope
approach as `buildVoice` (`index.html:887-941`): a short sine/triangle blip
with a fast attack (~1ms) and a ~30-50ms exponential decay to silence.

### Scope: no accent / no time signature

Clicks are uniform — no downbeat accent, no time-signature concept. Tonik
has no existing notion of beats-per-measure anywhere in the app, and none
was requested. BPM in, evenly-spaced clicks out.

## State & persistence

- `let bpm = 120` and `let metronomeOn = false` join the existing top-level
  state variables (`index.html:864-876`).
- BPM is persisted to `localStorage` under a new key (`tonik-bpm`), read on
  load and written on change — following the same pattern as the
  `tonik-finish` theme key (`index.html:1462`, `1466`).
- On/off state is **not** persisted. The metronome always starts off on
  page load, regardless of the last session.

## Out of scope

- Time signature / beat accents.
- Syncing the metronome to recorded track playback or to the recorder's
  count-in.
- Visual beat indicator beyond the toggle button's active state.
