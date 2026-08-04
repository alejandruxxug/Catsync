# catsync

Sprite sheet + audio → lip-synced animation, ready to drop over footage in your editor.

### → [alejandruxxug.github.io/Catsync](https://alejandruxxug.github.io/Catsync/)

Or download [`catsync.html`](https://github.com/alejandruxxug/Catsync/releases/latest) and double-click it. Same app either way — one file, vanilla JS, no dependencies, no build step, no server. Nothing you load ever leaves your machine; there's no network code in it at all.

Built for the cats in a class video about compilers, which is why frame 0 is "mouth closed" and everything is pixel-art shaped. It works on any sprite sheet.

Repo layout: `catsync.html` is the whole app. `index.html` is just the landing page GitHub Pages serves at the root.

## Use

1. **Load demo** to see the whole pipeline work before you have real art.
2. Drag in your talking sprite sheet. It guesses the grid; fix it in **Slicing** if the guess is off.
3. Hit **Record**, or drag in a WAV or MP3.
4. Hit Play, tune, export.

## Recording

Hit **Record** and talk. The take goes straight into the pipeline, the level meter goes amber near clipping and red past it, and **Save take (.wav)** gets it onto disk — a recording lives in the tab and nowhere else until you do, which is why leaving the page asks you to confirm.

Three settings are explicitly turned **off** on the mic, and this matters more than it sounds:

- **Auto gain control** normalizes loudness over time. Loud-against-quiet *is* the lip sync signal, so AGC is actively erasing what's being measured — it's the one that ruins a take.
- **Noise suppression** gates room tone down to digital silence. The threshold is a hard gate placed relative to the noise floor, and a floor of exactly zero turns it into a cliff with nothing sensible to sit on.
- **Echo cancellation** is tuned for calls and colours the voice.

The mic is wired to the level meter and to nothing else — it never reaches your speakers, so there's no feedback loop and you don't need headphones.

**Takes are re-encoded to WAV before anything else touches them.** MediaRecorder writes a live-stream container with no length in its header — the same hole that made the WebM export come out short. An `<audio>` element handed one of those reports `Infinity` for duration and refuses to seek, which breaks the waveform and the scrubber. The samples are already decoded by that point, so writing a plain WAV with a real header costs nothing and sidesteps the entire class of bug. Mono, since the analysis reduces to mono anyway.

Recording needs a secure context. The hosted page is HTTPS so it's fine; from `file://` Chrome and Firefox allow it and Safari doesn't. Takes are capped at 10 minutes.

Frames must be ordered **quietest to loudest** — frame 0 is mouth closed, last frame is widest. Any count from 2 up works.

## Slicing

A sheet is not always `W/cols × H/rows`. Generated ones come back with a margin around the outside, gutters between cells, or a size that just doesn't divide evenly — and refusing all three, which is what the old slicer did, meant reaching for a pixel editor before you could start.

The grid is its own thing now: an **origin**, a **cell size**, and a **gutter**. Cols × rows only says how many cells to step through. The sheet is drawn with the grid on top, everything outside it dimmed and each cell numbered — drag the grid to move it, drag the pink handle on the first cell to resize every cell, or arrow-key the origin once the sheet is focused.

Which control does what:

- **Cols, rows, gutter** refit the cells to the sheet. Whatever the margin and gutters leave over gets split evenly, so a grid never quietly runs off the edge because you asked for one more column.
- **Cell W/H and origin** are taken exactly as typed. Dragging never resizes; resizing never moves.

A grid that overhangs the sheet is allowed — the cells that fall outside come back partly empty rather than erroring, which is how you add padding. You get told when the grid overhangs, and when part of the sheet isn't covered.

**Auto-detect** reads the grid off the sheet instead of asking for it, and runs automatically on any sheet you drop. A column of pixels that's entirely background — key colour or already transparent — can't be inside a sprite, so the content runs between those columns are the cells; same for rows. It sets cols and rows too.

Two things it has to get right:

- A gap *inside* a sprite, between two legs say, would otherwise read as a cell boundary. Real gutters are the widest gaps on the sheet and roughly uniform, so anything under half the widest gap is treated as part of the sprite. No fixed pixel threshold — it scales with the sheet.
- Cell spacing comes from the first-to-last content run, which averages out the per-cell drift that generated sheets always have. Origin and cell size are then picked so every run fits inside its cell, so nothing gets clipped even when the drift is bad. If content is so uneven that cells would overlap, they're clamped to the spacing and you're told to check for a neighbour bleeding in.

If the sprites touch with no gutter at all it finds one blob, and falls back to trimming the outer margin and dividing that by the cols you asked for. If the whole sheet reads as background, the key colour is wrong — fix it in **Look** and detect again.

The demo loads a deliberately even grid, because auto-detect would trim it down to the sprites and hide the drift the align panel exists to fix.

## Align — onion skin

Image models can't hold a sprite pixel-identical across cells, so generated sheets come back with each frame shifted a little. Fix it here instead of round-tripping through a pixel editor.

Click any frame in the strip to select it. The reference frame is ghosted underneath — drag the sprite, or focus the canvas and use arrow keys (shift = 4px). Offsets are whole pixels and apply to the preview and every export.

- **Ghost / Difference** — Difference blend makes a misalignment obvious; anything still lit up is off.
- **Reference** — frame 0 (everything anchors to one frame) or previous (chains down the strip).
- **Auto-align** — searches for the shift that best matches the reference, weighting the silhouette heavily since the mouth is the one part allowed to differ.

The search window is 6% of sprite size, and on anything above 192px the matching runs on a downscaled copy — otherwise the cost, which is (2R+1)² × pixels, freezes the tab on a high-resolution sheet. Precision ends up relative rather than absolute: exact at 48px, within 4px at 1024px, which is 0.4% and invisible once composited. Measured 44–108ms per frame from 48px to 1024px. Arrow keys give you exact control if you want it.

**Every frame can be moved, frame 0 included.** It's still the alignment reference, but locking it meant a sprite clipped by the viewport edge could never be pulled back into frame. Tick **Move all frames together** to reframe the whole set without disturbing the relative alignment between frames — auto-align keeps working afterwards, since it positions each frame relative to wherever the reference currently sits. Offsets are clamped to one cell in each direction so a runaway drag can't lose a sprite off-screen.

The demo sheet ships with deliberate drift on cells 1–3 so you can see the feature work.

## Viewport

The exported frame is a window onto the cell, not the whole cell — trim dead space or reframe to head-and-shoulders. Set X/Y/W/H in sprite pixels, or hit **Fit to content** for the union bounding box of every frame's opaque pixels with 1px padding. It's outlined in teal on the align canvas, with everything outside dimmed.

**The cut isn't applied while you edit.** Until you tick **Lock the cut**, the preview shows the whole frame with the viewport drawn as a guide — dimmed outside, teal outline — so a sprite you're dragging stays visible even when it currently falls outside the crop. Lock it to see the real cut. Either way, **exports always apply the viewport**; the lock only changes what the preview shows. The **Output** stat always reports the exported size, never the preview's.

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
- **The encoder can fall behind realtime**, badly, on a large canvas — frames queue inside it and `stop()` discards whatever hasn't been written, silently cutting seconds off the end. Wall-clock time can't see this, so progress is now read out of the stream itself: every Cluster carries an absolute Timecode, and the highest one seen is how far encoding has genuinely got. Recording doesn't stop until that catches up to the target (with a stall detector and a hard cap so it can't hang). Bitrate scales with frame size instead of a flat 8Mbps, and VP8 is preferred over VP9 since it encodes much faster in realtime. If it still comes up short you get told exactly how short and why, rather than a quietly truncated file.
- The encoder needs a moment to flush after the last frame, and going silent during that window leaves recorded time with no frames in it, which players render as a held frame. That produced a ~0.33s freeze at the end of every clip — absolute, not proportional, so it read as "the animation stops just before the end." The final frame is now re-emitted throughout the flush, and the duration written into the container is the length actually measured rather than the theoretical one.

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

## License

MIT — see [LICENSE](LICENSE). Take it, change it, ship it.
