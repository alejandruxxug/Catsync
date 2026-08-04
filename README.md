# catsync

Sprite sheet + audio → lip-synced overlay, for the CRAFTING CODE video cats.

Double-click `index.html`. That's it — no server, no install, no internet. One file, vanilla JS, works from `file://`.

## Use

1. **Load demo** to see the whole pipeline work before you have real art.
2. Drag in your talking sprite sheet, set columns × rows (the Pass 2 sheet is `4 × 1`).
3. Drag in a WAV or MP3.
4. Hit Play, tune, export.

Frames must be ordered **quietest to loudest** — frame 0 is mouth closed, last frame is widest. Any count from 2 up works.

## Align — onion skin

Image models can't hold a sprite pixel-identical across cells, so generated sheets come back with each frame shifted a little. Fix it here instead of round-tripping through a pixel editor.

Click any frame in the strip to select it. The reference frame is ghosted underneath — drag the sprite, or focus the canvas and use arrow keys (shift = 4px). Offsets are whole pixels and apply to the preview and every export.

- **Ghost / Difference** — Difference blend makes a misalignment obvious; anything still lit up is off.
- **Reference** — frame 0 (everything anchors to one frame) or previous (chains down the strip).
- **Auto-align** — searches for the shift that best matches the reference, weighting the silhouette heavily since the mouth is the one part allowed to differ.

The search window is 6% of sprite size, and on anything above 192px the matching runs on a downscaled copy — otherwise the cost, which is (2R+1)² × pixels, freezes the tab on a high-resolution sheet. Precision ends up relative rather than absolute: exact at 48px, within 4px at 1024px, which is 0.4% and invisible once composited. Measured 44–108ms per frame from 48px to 1024px. Arrow keys give you exact control if you want it.

Frame 0 is the anchor and can't be moved. The demo sheet ships with deliberate drift on cells 1–3 so you can see the feature work.

## Viewport

The exported frame is a window onto the cell, not the whole cell — trim dead space or reframe to head-and-shoulders. Set X/Y/W/H in sprite pixels, or hit **Fit to content** for the union bounding box of every frame's opaque pixels with 1px padding. It's outlined in teal on the align canvas, with everything outside dimmed.

The viewport applies after alignment, so align first, then frame.

## Display vs export size

**Fit preview to window** (on by default) scales the preview down to fit — display only, it never touches what you export. The **Output** stat under the preview always shows the real exported pixel size, which is the viewport dimensions × **Scale**. If the exported PNGs come out a size you didn't expect, that readout is the one to trust.

## Exports

**PNG sequence (.zip)** — the real one. Numbered transparent PNGs, import into Premiere / Resolve / CapCut as an image sequence. Exact alpha.

**WebM** — no transparency. Browsers can't record an alpha channel, so the key color is baked in as a solid background for you to chroma-key. Records in real time, so a 60s clip takes 60s.

Three things had to be handled to get the length right:

- MediaRecorder writes a *live-stream* WebM — the Segment has unknown size and Info carries no Duration element, since the length isn't known when recording starts, and nothing backfills it on stop. Players then guess from cluster timecodes and often guess low, which is what makes clips look short. We know the exact length, so it's written into the container afterwards. If the file has a SeekHead or Cues, inserting bytes would invalidate their offsets, so the patch is skipped and you get a warning instead of a broken file.
- Frames are emitted explicitly via `requestFrame()` on a wall-clock timer. On auto-capture the canvas is only sampled when it happens to be dirty, and rAF throttles to ~1fps in a background tab, so the cadence silently collapses.
- Recording now arms before playback starts, with a primed first frame — previously the gap between `play()` resolving and the recorder starting was lost off the front.

Still, the PNG sequence is the dependable export. Use WebM for quick checks.

## The three settings that matter

**Silence threshold** — auto-set from each new file's noise floor, which is the only sane default since it depends on your mic and room. It's a hard gate, so it flips suddenly rather than sliding: on a test clip, 18 → 20 moved mouth-closed from 3% to 40%. Nudge in small steps.

**Min hold** — how many frames a mouth shape must stay before it can change. 2 at 30fps is a good start. Too low, the mouth strobes. Too high, it lags the words. On the demo, raising hold from 1 to 4 cut mouth changes from 37 to 20.

**Mouth closed %** — the readout that tells you if the other two are right. A normal take of continuous speech should land somewhere around 30–40%. If it reads 2%, the mouth never closes between sentences and your threshold is too low.

## How the sync works

Amplitude-driven, not phoneme-driven. Loud = open, quiet = closed.

Audio is downmixed to mono, split into one window per video frame, and each window's RMS is converted to dB. That's normalized against the **95th percentile** of non-silent frames rather than the peak, so one loud pop doesn't squash the rest of the take. The 0–100 result maps into equal bands, one per sprite. Hysteresis stops values sitting on a band edge from chattering; minimum hold stops strobing.

Real phoneme lip sync needs a transcript, and the cats are small on screen. This reads fine.

## Notes

- Changing threshold, hold, or scale only redoes the cheap mapping step — the audio is decoded and analyzed once.
- Scaling is nearest-neighbour only. Blurred pixel art is a failed render.
- Keying defaults to a hard cutoff, because soft edges look wrong on true pixel art. **Feather** (in Look) softens it for anti-aliased or high-resolution sprites — alpha ramps from the tolerance distance out to tolerance + feather.
- **Despill** removes the key-colour fringe that feathering exposes. It only touches partly-transparent pixels, scaled by how transparent they are, so opaque interior colours are never altered — a genuinely pink nose survives a magenta key. It desaturates toward each pixel's own mean rather than clamping the key's dominant channels: with magenta those are R and B, so clamping collapses toward the lone dark G channel and leaves a dark halo. Pivoting on the mean holds luminance exactly and just drains the colour cast.
- Record **one long WAV per script segment**, not per line. Fewer passes, and the normalization is steadier across a whole take.
