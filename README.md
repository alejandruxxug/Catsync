# catsync

Sprite sheet + audio → lip-synced overlay, for the CRAFTING CODE video cats.

Double-click `index.html`. That's it — no server, no install, no internet. One file, vanilla JS, works from `file://`.

## Use

1. **Load demo** to see the whole pipeline work before you have real art.
2. Drag in your talking sprite sheet, set columns × rows (the Pass 2 sheet is `4 × 1`).
3. Drag in a WAV or MP3.
4. Hit Play, tune, export.

Frames must be ordered **quietest to loudest** — frame 0 is mouth closed, last frame is widest. Any count from 2 up works.

## Exports

**PNG sequence (.zip)** — the real one. Numbered transparent PNGs, import into Premiere / Resolve / CapCut as an image sequence. Exact alpha.

**WebM** — no transparency. Browsers can't record an alpha channel, so the key color is baked in as a solid background for you to chroma-key. Records in real time, so a 60s clip takes 60s.

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
- Keying is a hard cutoff with no feathering, because soft edges look wrong on pixel art.
- Record **one long WAV per script segment**, not per line. Fewer passes, and the normalization is steadier across a whole take.
