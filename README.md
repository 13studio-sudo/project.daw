# PROJECT-DAW

A single-file, browser-based digital audio workstation. No build step. No
dependencies. Open `index.html` and play.

---

## Edition

**v0.6.0** — MVP edition · open source
*browser workstation · edition 0.0.6*

---

## What's in the box

- `index.html` — the entire app (~6,200 lines: HTML + CSS + vanilla JS)
- `scheme.html` — a visual scheme/architecture reference showing the project's
  audio chain, panel layout, and theming system
- `README.md` — this file

That's it. No `node_modules`, no bundler, no toolchain. Drop the folder
anywhere with a browser and double-click `index.html`.

---

## Quick start

1. Open `index.html` in a modern browser (Chrome, Firefox, Safari, Edge —
   anything from the last ~5 years with AudioWorklet support).
2. Click the loading veil that says `let's play`. This is the user gesture
   the browser requires before audio can start.
3. The workspace appears with one drum track and one synth track pre-seeded
   so you can hit anything immediately and get sound.

---

## Features

### Tracks
- Multi-track with two track types: **drum** and **synth**
- Per-track volume, mute, solo, arm-for-record
- Synth voices: raw waveforms (sine/triangle/square/sawtooth) plus
  preset voices (piano/rhodes/...) ported from `ear_trainer`
- 909-style synthesized drums (kick/snare/hat/perc) — no samples, built
  from oscillators + noise + envelopes

### Sequencer
- 4×16 step grid per drum track
- BPM control, looping (1/2/4/8 bars), metronome
- Transport: play / stop / record

### Performance
- Proportional piano keyboard (1–5 octaves, rebuilds on panel resize)
- KAOSS XY effects pad
- Computer keyboard input for live playing

### FX & master
- Per-bus filter, distortion (WaveShaper), **bitcrush (AudioWorklet)**,
  delay with feedback
- Master 3-band EQ (low shelf, peaking mid, high shelf)
- Master reverb (convolver, synthesized IR) and master delay sends

### Visualizers
- Oscilloscope with multiple modes
- 20×10 LED-style spectrum **Freq Matrix**

### Recording
- Master recorder exports the rendered mix as a downloadable audio file
  (WebM via MediaRecorder, or WAV via AudioWorklet PCM tap)
- Per-track note recording into the timeline

### Workspace
- Resizable, draggable panels in a fractional grid (no stale-pixel issues
  across viewport changes — the layout always recomputes on boot)
- Five themeable CSS variables drive the entire visual system; everything
  else is derived via `color-mix()`
- "RESET" in the toolbar and "RESET ALL" in settings share one
  implementation: clear theme + layout, keep tracks

---

## Changelog

### v0.6.0 (edition 0.0.6)
- **Panel rename + reorder**: rearranged the analyzer/effect panels
  to keep visualizers contiguous and put the XY effect pad at the end.
  | new | old | name                |
  |-----|-----|---------------------|
  | [05] | [06] | spectrum-scope (renamed, was "spectrum · scope") |
  | [06] | [07] | freq matrix         |
  | [07] | [05] | xy effect pad       |
  - All cosmetic; internal panel IDs (`pScope`, `pFreq`, `pKaoss`)
    unchanged. Layout engine, dropdowns, and saved state are unaffected.

### v0.5.0 (edition 0.0.5)
- **Freq matrix · JS-driven sizer**: replaced the CSS aspect-ratio
  approach with a `ResizeObserver`-driven sizer that computes integer
  pixel sizes for cells. Guarantees three invariants regardless of
  panel shape:
  1. every cell is exactly square,
  2. horizontal and vertical gaps are both exactly 2px,
  3. the matrix scales to fit the available space with letterbox /
     pillarbox space around it (Option A: never distort).
  The matrix now uses fixed pixel grid-tracks (`repeat(N, var(--fm-cell))`)
  instead of `fr` units. Sizing fires on every panel resize via the
  observer; build() and setSize() both trigger a fresh measurement.

### v0.4.0 (edition 0.0.4)
- **Panel renumbering**: panels are now numbered sequentially [00]–[07]
  with no gaps. Previously the spectrum scope was [09] and the freq
  matrix [10]; they're now [06] and [07] respectively. Panel IDs in
  the code (`pScope`, `pFreq`, etc.) are unchanged — this is purely a
  cosmetic update to the title bars.
- **Freq matrix · cell aspect-ratio lock**: every cell is now guaranteed
  square via two redundant CSS constraints — the container has
  `aspect-ratio: COLS / ROWS` (always 2:1 in both modes), and each
  `.fm-cell` has `aspect-ratio: 1 / 1` as a backup. Cells no longer
  stretch when the panel is resized awkwardly.
- **Freq matrix · 40×20 mode**: a new dropdown in the panel head lets
  you switch between **20×10 (200 cells)** and **40×20 (800 cells)**.
  Higher resolution gives a denser spectral view. Both modes share the
  same 2:1 container aspect ratio so the panel doesn't reshape.

### v0.3.0 (edition 0.0.3)
- **AudioWorklet migration**: replaced both `ScriptProcessorNode` uses with
  modern `AudioWorkletNode` processors. The previous implementations were
  deprecated and ran their DSP on the main thread.
  - `BitcrusherProcessor` — bit-depth quantization + sample-rate
    decimation. Exposes `bits` (1–16) and `normFreq` (0–1) as `AudioParam`
    descriptors so values can be automated. Uses symmetric quantization
    (`Math.round`) for clean handling of zero-crossings.
  - `WavTapProcessor` — captures stereo float32 samples from the master
    bus and posts them to the main thread via `port.postMessage` for WAV
    export. Replaces the old `onaudioprocess` callback.
  - Both processors are defined as JS string literals, blob-wrapped, and
    loaded via `audioWorklet.addModule()` from an object URL. This keeps
    PROJECT-DAW shipping as a single HTML file.
  - The bitcrusher exposes `.bits` and `.normFreq` accessor properties on
    the worklet node so the existing dispatcher (`Audio.bitcrushNode.bits
    = ...`) continues to work unchanged — backwards-compatible upgrade.
  - During the brief (~50ms) worklet load window, a passthrough gain holds
    the bitcrusher position in the FX chain so audio remains functional
    from `Audio.init()` onward. The real worklet node is spliced in once
    `addModule()` resolves.
- **No behavioral changes** — the bitcrusher sounds identical, recordings
  still produce the same WAV format. Internal architecture only.

### v0.2.0 (edition 0.0.2)
- **FIX**: Freq Matrix no longer lights up its lowest cells at idle. The
  root cause was the WaveShaper distortion node's lookup curve having
  an off-by-half-index asymmetry: with a 256-entry curve, input 0.0
  interpolates between `curve[127]` and `curve[128]`, and the original
  formula made those `-0.0078` and `0.0` respectively — so silent input
  produced `-0.0039` of DC, propagating through the FX chain (and getting
  amplified by the 0.42 feedback delay loop). Fixed by sampling the curve
  at `(2i+1)/n - 1` instead of `(2i)/n - 1`, which is symmetric around
  the half-integer midpoint: `curve[127] = -curve[128]`, interpolation at
  x=0 = exactly 0.
- **REFACTOR**: Toolbar `RESET` and settings `RESET ALL` consolidated into
  a single shared function `resetAllSettings()`. Both buttons produce
  identical behavior (clear theme + layout, keep tracks).
- **DEFENSIVE**: Time-domain silence gate added to Freq Matrix as a
  belt-and-suspenders measure.

### v0.1.8 (edition 0.0.1)
- Initial MVP release.

---

## Architecture in one paragraph

Single HTML file with three sections: `<style>` (theming via 5 CSS
variables + a 4px-module spacing scale), `<body>` (one panel per feature,
docked into a computed grid; plus an overlay layer for the `[00] Let's
Play` welcome screen), and `<script>` (state, audio engine, voices,
samples, drum grid, piano, transport scheduler, timeline, KAOSS pad,
master recorder, auto-layout engine, oscilloscope, freq matrix, settings).
The audio engine runs entirely in modern Web Audio: BiquadFilters,
WaveShaper, AudioWorkletProcessors (bitcrusher + WAV tap), Convolver
reverb, and Gain/Delay nodes. Boot sequence: load theme → build piano →
wire UI → init AudioContext (kicks off async worklet load) → seed two
default tracks → apply grid layout → start visualizers.

See `scheme.html` for the visual diagram.

---

## Browser compatibility

Tested on:
- Chrome / Edge (Chromium)
- Firefox
- Safari (desktop)

AudioWorklet has been universally supported since 2018 (Safari 14.1 in
2021 was the last to ship it). Anything from the last ~5 years works.

Mobile browsers will load but the UI is desktop-first (no touch-optimized
panel dragging yet). Web Audio works in mobile Safari but requires the
user gesture for AudioContext.resume(), which the veil already provides.

---

## Tech notes for hackers

- **Theming**: edit `--bg`, `--ink`, `--ink-dim`, `--line`, `--line-soft`
  in the `:root` block — the rest of the palette is derived via
  `color-mix(in srgb, …)`. Or just open Settings and use the theme picker.
- **No state persistence besides settings**: tracks are session-only.
  Pixel-position panel state is intentionally NOT persisted because saved
  pixels go stale across browser-width changes; only collapse state is
  restored from localStorage.
- **AudioContext is shared**: a single `Audio.ctx` runs the whole app.
  `Audio.fxIn` is the input to the master FX chain that all tracks route
  into.
- **AudioWorklet processors are defined as JS string literals** at the top
  of the audio module. They're blob-wrapped and loaded via
  `addModule(blobURL)` so the project stays single-file.
- **Worklet load is async but the chain is synchronous**: the bitcrusher
  position is held by a passthrough gain until the real worklet node
  loads, then spliced in. The dispatcher writing `bitcrushNode.bits = ...`
  works in both states (passthrough has stamped properties; worklet has
  accessor properties routing through AudioParam).

---

## License

Open source. Use it, fork it, ship something with it.

