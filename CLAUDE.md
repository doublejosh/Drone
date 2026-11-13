# Drone Synth

A browser drone synthesizer: pick a key and mode, and it sustains a root-based chord through a filtered, LFO-swept, reverberated Web Audio graph. It shows a live oscilloscope.

The whole app is one file, [index.html](index.html), with inline CSS and JS. It has no build step, no dependencies and no tests.

## Running

Open `index.html` in a browser, or serve the folder (`python3 -m http.server`). Audio starts only after a user gesture (Play button or Space), because browsers require one before an `AudioContext` can run.

To check a change, load the page, press Play, and exercise the control you touched. Watch the scope and listen for clicks or zipper noise.

## File layout (index.html)

| Lines (approx) | Section | Notes |
|---|---|---|
| `<style>` | Theme + layout | Colors are CSS vars on `:root` (`--bg`, `--panel`, `--accent`, `--accent-2`…). Dark-only. Breakpoint at 560px shortens the piano and the Play button. Key picker is a one-octave piano (`.piano`): white keys are flex items, and black keys are absolutely positioned. Each black key's `left` is computed in `buildUI()` from `BLACK_AFTER_WHITE` and the `--gap`/`--bw` CSS vars. Black keys are out of flex flow, so DOM and tab order stay chromatic (C, C♯, D…). |
| `<main>` markup | Static shell | `.display` (key name, Hz, chord, Play button, `<canvas id="scope">`), then five `<section>`s: **Key**, **Slots**, **Tone**, **Shape**, **Ambience**. Containers `#notes`, `#octaves`, `#slots`, `#waves`, `#sliders`, `#ambience` are filled by JS. Mode and voice chips are hard-coded in HTML. |
| `NOTES`, `WAVES`, `VOICES`, `state`, `SLIDERS` | Config + state | The single source of truth. Most features start here. |
| Music helpers | `midiOf`, `hz`, `thirdOffset`, `voiceOffset` | MIDI math; A4 = 440 Hz; octave numbering is scientific (C4 = MIDI 60). |
| Audio engine | `initAudio`, `applyPitch`, `applyParams`, `setPlaying`, `makeImpulse` | Graph is built once, lazily, on first play. |
| Ambience | `SOUNDS`, `AMB_BUFFERS`, `makeLoop`, `applyAmbience`, graph helpers (`chain`, `gainN`, `filt`, `lfoOsc`, `mod`, `loopSrc`) | Procedural nature-sound layers. See **Ambience** below. |
| UI | `buildUI`, `render` | `buildUI` wires events once; `render` syncs labels and `.active` classes from `state`. |
| Oscilloscope | `draw` | `requestAnimationFrame` loop reading `analyser` time-domain data, triggered on a rising zero crossing. |

## State model

```js
state = {
  note: 0–11 (index into NOTES, C = 0), mode: 'major'|'minor', octave: 1–4, wave: one of WAVES,
  voices: { root, sub, third, fifth, up }  // booleans
  detune, cutoff, lfoRate, lfoDepth, reverb, glide, waveFade, volume,  // slider-backed numbers
  playing: bool,
  slot: index, slots: [{ note, mode, octave }, ...],  // key slots, 1..MAX_SLOTS (12)
  auto: bool, bpm: BPM_MIN..BPM_MAX (1–240),          // slot auto-advance
  ambience: { [soundKey]: { on: bool, level: 0..1 } }, // one entry per SOUNDS key
}
```

**Key slots:** `state.note`, `state.mode` and `state.octave` always hold the *active* slot, so all existing controls edit it unchanged. `state.slots[]` is authoritative only for *inactive* slots, so call `saveSlot()` before reading or reordering `slots` if the active one matters.
- `selectSlot(i)` runs `saveSlot()`, then `loadSlot(i)`, which calls `applyPitch()`, so a switch glides like any other pitch change.
- **Shift** steps to `(slot + 1) % length` (key repeat is ignored), and clicking a chip jumps to that slot.
- `addSlot()` inserts a copy of the active slot right after it and selects the copy. `removeSlot()` deletes the active slot and loads the slot that slides into its place. A minimum of 1 slot is enforced, and the +/− buttons disable at the limits.
- The slot chips in `#slots` are rebuilt on every `render()`.
- **Auto mode** (`#autoBtn`, `setAuto`, `scheduleAuto`): a `setTimeout` chain calls `nextSlot()` once per beat (`60000 / bpm` ms), but only while `state.playing`. Each step is timed from `lastStep`, so `setBpm()` re-times the *pending* step rather than restarting the countdown; dragging the slider doesn't stall it. Pressing Play resets `lastStep`, so the first step lands one beat later. The timing is wall-clock and not sample-accurate, which is fine for slow chord changes. For tight rhythm, schedule against `ctx.currentTime` instead.
- **Tempo UI:** `#bpmRange` and `#bpmInput` mirror each other. The number box applies valid values as you type and clamps or reverts on `change`. Both live in the Slots panel, not in `SLIDERS`, so they're wired by hand in `buildUI()`. The global keydown handler ignores events from `type=number` inputs so typing there doesn't trigger shortcuts.
- To add more per-slot fields (e.g. wave), include them in `saveSlot()` and the `slots` defaults. `loadSlot` uses `Object.assign`, so it picks them up automatically. Then call `applyParams()` in `loadSlot` as well.

The update pattern every control follows is **mutate `state` → call `applyPitch()` and/or `applyParams()` → call `render()`**.
- `applyPitch()`: anything that changes oscillator frequency (note, mode, octave). It respects `state.glide`.
- `applyParams()`: everything else (filter, LFO, reverb mix, voice gains, detune, wave type, master volume while playing).
- Both are no-ops before `ctx` exists, so they're safe to call anytime. `initAudio()` applies the current state when it runs.

## Signal graph

```
for each voice in VOICES, for each wave in WAVES (all always running)
  osc(-) ─┐                                       (detuned ±state.detune cents)
  osc(+) ─┴─> wave gain (1 if active wave, else 0) ─┐
              (one per wave) ───────────────────────┴─> voice gain (VOICES[v].level or 0) ─┐
                                                  ├─> mix (0.22, headroom) ─> lowpass filter (Q 2)
                                                  │                              ▲ frequency
                         lfo (osc) ─> lfoGain ────┼──────────────────────────────┘
filter ─┬─> dry ───────────────┐
        └─> convolver ─> wet ──┴─> master (volume / 0 when stopped) ─> analyser ─> destination
```

Key behaviors:
- **Every voice oscillates all the time.** Turning a voice on or off only ramps its gain (time constant 0.3s). That keeps toggles click-free and phase-continuous.
- **Stop** ramps master to 0 and then suspends the context after 4s if it's still stopped. **Play** resumes it and ramps master up (τ 0.5s).
- **The LFO modulates `filter.frequency`.** Depth is clamped to `cutoff * 0.95` so the swept frequency never goes negative.
- **Reverb** is a convolver with a generated stereo noise impulse (4s, decay exponent 2.5). Dry gain is `1 - reverb*0.7` and wet gain is `reverb*1.4`.
- **Glide:** `setTargetAtTime(f, t, glide/3)`, so it reaches about 95% of the target in `glide` seconds.
- **Tone (wave) crossfade:** every waveform runs continuously per voice (5 voices × 4 waves × 2 oscs = 40 oscillators). Switching Tone ramps the wave gains with τ = `max(0.01, waveFade/3)`, so it's seamless. All native waves have their fundamental in phase, so the midpoint of a crossfade doesn't hollow out. `voiceNodes[name].oscs` is flat, ordered `[-, +]` per wave, so detune sign is `i % 2 ? 1 : -1`.
- **Param smoothing:** time constant 0.08s (`tc`), or 0.001s on init.

## Ambience

The nature-sound layers are all **synthesized**, with no audio files:

```
SOUNDS[key].build(out) graph ─> per-sound gain (level², 0 when off; τ 0.4s) ─> ambOut ─> destination
```

- **`ambOut` is gated with `master`.** Play/Stop and Volume ramp both buses together, in `setPlaying` and at the end of `applyParams`. Ambience bypasses the drone's filter, reverb and the analyser, so the scope shows only the drone.
- **Lazy build.** `applyAmbience()` builds a sound's graph the first time it's enabled, which requires `ctx`, so a toggle before the first Play only sets state. The graph then runs forever and is muted when off. `applyParams()` calls `applyAmbience()`, so `initAudio()` builds any sounds already enabled.
- **Buffers** (`AMB_BUFFERS`, cached in `ambBufs`) are stereo loops made by `makeLoop(seconds, gen)`. `gen(n, rate)` returns `[L, R]`, and the extra 0.25s tail is equal-power crossfaded into the head so the loop doesn't click. Loop lengths are 5/7/9/11/13 s so layers don't phase-align. `loopSrc()` starts each source at a random offset.
  - `white`, `brown`: noise beds.
  - `drops`: sparse decaying sine pings (rain).
  - `crackle`: noise pops with occasional bursts (fire).
  - `crickets`: 4 panned chirping insects.
- **Design recipes:**
  - Rain: hiss, low body and drops.
  - Wind: band-passed brown noise with slow LFOs on its frequency and gain (gusts), plus a resonant whistle.
  - Waves: two swell layers at unrelated rates. One LFO per layer drives loudness, brightness and foam together.
  - Stream: 4 resonant band-passes with faster LFOs (babble).
  - Fire: flickering low roar plus crackle.
  - Crickets: the chirp buffer, high-passed.

### Add an ambient sound
1. Add `key: { label, icon, build(out) { … } }` to `SOUNDS`. Add `icon` to the font `<link>`'s `icon_names` too. Wire the graph into `out` with the helpers: `chain(a, b, c, out)`, `filt(type, f, Q)`, `gainN(v)`, `mod(lfoOsc(rateHz), depth, param)`, and `loopSrc(ambBuf('white'|'brown'|…))`.
2. If it needs a new buffer, add a generator to `AMB_BUFFERS` that uses `makeLoop`. `addPing` and `addPop` place one-shot events.
3. Add `key: { on: false, level: 0.5 }` to `state.ambience`.

The toggle and level slider are generated by `buildUI()`. Internal gains are set by estimate, not by ear, so balance a new sound against the others at the same slider level.

## Common changes, step by step

### Add a slider parameter
1. Add a default to `state` (e.g. `resonance: 2`).
2. Add an entry to `SLIDERS`: `{ key, label, icon, min, max, step, fmt, log? }`. Add `icon` to the font `<link>`'s `icon_names` too (see **Icons**).
   - The `<input>` always runs 0–1000 internally. `toVal` and `toPos` map that range to the real value.
   - Set `log: true` for frequency- or time-like ranges. `min` must be > 0 when `log` is set.
3. Use `state.<key>` inside `applyParams()`, and smooth it with `setTargetAtTime(…, t, tc)`.
4. If it needs a new node, create it in `initAudio()`, store it in the module-level `let` list, and splice it into the graph.

The slider UI is generated automatically.

### Add a voice (e.g. 7th, 2nd octave up)
1. Add it to `VOICES` with `offset` (semitones from root) and `level`. Use `offset: null` plus a branch in `voiceOffset()` if it depends on the mode, the way `third` does.
2. Add a default to `state.voices`.
3. Add a chip in the Tone section's HTML: `<button class="chip toggle" data-voice="name">Label</button>`.

The existing loops handle node creation, pitch, gain and active-state rendering.

Watch the total level as you add voices. The `mix` gain of 0.22 is the headroom budget, so lower it, or the voice levels, if clipping shows up on the scope.

### Add a waveform
Append it to `WAVES`. `initAudio()` automatically builds an osc pair and a wave gain for it in every voice, and the crossfade picks it up. Native types are `sine`, `triangle`, `sawtooth` and `square`. A custom wave needs `ctx.createPeriodicWave` plus `o.setPeriodicWave()` in the `initAudio()` loop in place of `o.type = w`. Keep its fundamental in phase (positive sine term) so crossfades stay full. Each wave adds 10 oscillators (5 voices × 2).

### Add a mode/scale (e.g. sus2, sus4, dorian flavor)
`state.mode` is currently binary, and these places assume that:
- `thirdOffset()` (major → 4, minor → 3)
- the mode chips in the HTML (`data-mode`)
- the `M` shortcut, which toggles between the two
- `#randomBtn`, which picks between the two
- `render()`'s chord display `[0, thirdOffset(), 7]`, plus the `<small>` label

Generalize it with a `MODES` table, e.g. `{ minor: { third: 3, fifth: 7 }, sus4: { third: 5, fifth: 7 } }`, and read from it everywhere listed above.

### Add a keyboard shortcut
Add a branch in the `keydown` listener inside `buildUI()`. A branch that doesn't match must reach the final `else return` so it won't swallow other keys. Arrow keys are ignored while a range input has focus. Update the `.hint` `<p>` text as well.

### Change the octave range
`[1, 2, 3, 4]` is hard-coded in `buildUI()`, and `Math.min(4…)` / `Math.max(1…)` is hard-coded in the keydown handler. Change both, or lift them into a constant.

### Presets / save & load / URL state
`state` is plain JSON-able data, with `playing` being the only exception. If you add a feature that sets slider-backed values in code, note that **`render()` does not update slider inputs or their `<output>`s**: those are written only once, in `buildUI()`. Factor the `input.value = toPos(...)` and `out.textContent = ...` lines into a sync function, then call it after loading state.

## Gotchas
- **Voices that match the wave's own harmonics get masked.** Sawtooth and square already contain strong overtones, so a quiet `+12` voice is nearly inaudible on them (`up` was once at 0.25 and seemed broken). Give overtone-aligned voices enough level, and test new voices on both sine and sawtooth.
- **Don't swap `o.type` on a running oscillator.** Doing so changes the waveform mid-cycle and clicks, which is why waves crossfade between dedicated oscillators.
- **Waves differ in loudness.** Square is noticeably louder than sine, and the crossfade doesn't normalize that. If needed, add a per-wave level to the wave-gain target.
- **The chord readout doesn't follow the voice toggles.** `#chordTones` always shows root, third and fifth, whichever voices are enabled.
- **`applyPitch` reads the in-flight frequency.** It uses `o.frequency.value` as the glide start point, so a glide that is interrupted continues from where it was.
- **`render()` updates `document.title`** with the current key.
- **Ambience buffers are generated on the main thread.** Each takes about 0.1–0.3s the first time a sound needs it (Rain needs three). The UI hitches briefly but the audio doesn't glitch. If you add many sounds, consider pre-generating buffers in `initAudio()` or in an idle callback.
- **No persistence.** A reload returns every setting to the defaults in `state`.

## Icons
Most labels are icons, with the text kept as `title` tooltips and screen-reader names.
- **Material Symbols Rounded** is loaded from Google Fonts by the `<link>` in `<head>`. It's **subset with `icon_names=`**, so:
  - **any new icon name must be added to that list**, which **must stay alphabetical**, or it renders as raw text (e.g. "volume_up");
  - check names against https://fonts.google.com/icons.
  - Markup: `<span class="ms" aria-hidden="true">icon_name</span>`. Give the button or label an accessible name with `aria-label`, `title`, or a `.sr` (visually hidden) span. Row labels use `<span class="label ms" role="img" aria-label title>`.
- Icons are outlined by default and switch to filled when their button is `.active` (and on the Play button when it's playing), via `font-variation-settings`.
- **Data-driven icons:**
  - `SLIDERS[].icon`, shown beside the visible `label` text. Shape sliders keep their text labels on purpose;
  - `SOUNDS[key].icon`, shown beside the visible `label` text on the Ambience toggles;
  - waveform buttons use inline SVG paths in `WAVE_PATHS` (viewBox 0 0 24 14), since Material Symbols has no waveform glyphs.
- **Kept as text on purpose:** Shape slider labels and Ambience toggles (icon + text), note names, Major/Minor, octave numbers, voice intervals (Sub/3rd/5th/Octave ↑), slot chips, section headings and the BPM unit.
- If the font fails to load (e.g. offline), icons degrade to their ligature names as text. The app still works.

## Conventions
- Keep everything in the single `index.html` unless a feature clearly justifies splitting it out.
- Write vanilla JS, with no frameworks or CDN libraries. The only external resource is the Material Symbols font. Use `$` for `querySelector`.
- Build UI elements data-driven from config arrays/objects when possible (see `NOTES`, `WAVES`, `SLIDERS`).
- Every audio parameter change must be smoothed (`setTargetAtTime`), never assigned directly while playing.
- Use `chip` buttons for choices and `chip toggle` buttons for on/off (purple when active).
