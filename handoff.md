# Handoff — O2 button microinteractions

_Last updated: 2026-06-24_

## TL;DR

A working prototype of animated **Reply / Follow / Like** flair buttons for the O2 / WordPress.com post footer, driven by After Effects → Lottie exports, presented in a modal. Open `index.html`, hover/click the buttons. It's deployed to a **private** repo: `github.com/nick-a8c/o2-button-microinteractions`.

Everything is self-contained — the only external dependency is `lottie-web` from a CDN.

---

## Status: what works

- **Reply** (`reply_i1`, top of modal) + **`reply_i2`** / **`reply_i3`** (stacked) — hover-only: play hover-on and hold the last frame while hovered; play hover-off on leave.
- **Follow** (`follow_i1`) and **Like** (`star_i1`) — full toggles: hover-on/off, click-to-select (Follow → blue box, Like → filled blue star), click-again-to-deselect, plus distinct hover animations while active.
- **Press feedback** — every button scales to 96% on `:active`.
- **Modal** — backdrop + card + close ✕, dismiss via ✕ / backdrop-click / `Esc`, reopen via "Open dialog".
- Icons are size-normalized (~22px) and cropped to the button's icon slot.

All segment timings were dialled in by hand with the segment-mapper (below) and verified frame-by-frame.

---

## Repo contents

| File | Purpose |
|---|---|
| `index.html` | The demo modal. **Generated** — do not hand-edit; edit `build_states.py` and rebuild. All 6 Lottie JSONs are inlined. |
| `segment-mapper.html` | Tool to scrub each Lottie, read exact frames, mark per-state start/end, and export the segment map as JSON. Also generated. |
| `build_states.py` | Source of truth for `index.html`: per-button config (segments, hold frames, viewBox crops, icon sizing) + the layout + the state-machine JS. **Re-run `python3 build_states.py` after any edit.** |
| `build_mapper.py` | Source for `segment-mapper.html`. |
| `states/*.json` | Raw AE Lottie exports: `reply_i1–i3`, `follow_i1`, `star_i1` (+ `reply-all`, unused in the layout but kept). |

---

## How a button is built

Each button is the **exact O2 button SVG** (rounded box + border + text label, copied as paths) with a `position:absolute` `.ico` span overlaid on the icon area. The Lottie loads into that span; its `<svg>` `viewBox` is then overridden to **crop** to just the icon (the comps are large and the icon is off-centre). A single generic state machine (`wire()` in `build_states.py`) drives all of them.

### Segments (absolute comp frames)

| Button | ip | hover-on | press | hoverOff·active | hoverOn·active | unpress | hover-off |
|---|---|---|---|---|---|---|---|
| reply_i1 | 16 | 43–74 | — | — | — | — | 84–89 |
| reply_i2 | 16 | 44–79 | — | — | — | — | 84–109 |
| reply_i3 | 15 | 40–64 | — | — | — | — | 84–105 |
| follow_i1 | 40 | 54–62 | 82–113 | 116–117 | 118–120 | 144–163 | 164–166 |
| star_i1 | 0 | 18–80 | 92–129 | 127–128 | 129–130 | 138–159 | 160–164 |

After each segment plays, the button **holds that segment's last frame** (the resting pose) until the next interaction.

---

## Gotchas / lessons (read before changing anything)

1. **Lottie frame base — the big one.** `playSegments([s,e], true)` uses **absolute** comp frames _and mutates_ `anim.firstFrame` to `s`. `goToAndStop(f, true)` is relative to the **current** `firstFrame`. So to rest on an absolute frame you must use `goToAndStop(absFrame - anim.firstFrame, true)` reading `firstFrame` **live** — never a captured `ip`. Getting this wrong makes icons land on the wrong frame and "disappear" after one play. (This bit us hard; `reply_i1` was the only button that masked it, because its art is in a fixed mask window that looks the same at every frame.)

2. **`ip ≠ 0` comps.** `follow_i1` has `ip=40`; its segment numbers as the designer marked them were **relative to ip** (had to add 40). Always sanity-check: a segment number `< ip` means it's ip-relative.

3. **Hold frames must be resting poses.** Don't blindly hold the literal last frame of a marked range if that frame is mid-transition (e.g. an arrow flown off-crop) — it'll look blank. The current ranges were re-marked so their ends are clean.

4. **Crops via `viewBox`, not CSS transforms.** Sizing/cropping is done with the Lottie SVG's `viewBox` + the `.ico` container box. Avoid `transform: scale()` on the icon — it rasterizes the SVG and blurs it.

5. **Icon sizes vary per comp.** Each animation has the icon at a different intrinsic scale, so per-button `ico` container overrides (in `build_states.py` `BUTTONS`) normalize them. `reply-all` is internally inconsistent (tiny idle chevron, large pressed arrow) which is why it was dropped from the layout.

---

## The segment-mapper tool

If timings need adjusting, open `segment-mapper.html`: each Lottie gets a scrubber + exact frame readout (`ip · op · fps`). Scrub to a boundary, click **⇐ set / set ⇒** on the matching state row, toggle **Hover only** for files with just hover on/off, then **Export** the JSON. Paste that back into `build_states.py`'s `BUTTONS` config (the `seg`/`hold` dicts) and `python3 build_states.py`.

---

## Open items / possible next steps

- **Top "Reply"** is `reply_i1` (a hover-only variant), so it doesn't have press/active states like Follow/Like. If a full reply toggle is wanted, a cleaner `reply-all` export (consistent idle vs pressed icon scale) would be needed.
- **`reply_i2` hover-off** ends around frame 109 where the arrow is extended; verify it reads well, or trim the range.
- **Colours** vary by export (some idle arrows are gray, some dark; hover accents are blue). Not unified — confirm intended.
- No real wiring to O2 yet — this is a standalone prototype. Productionizing means replacing the inlined JSON approach with proper asset loading and hooking the state machine to the real button markup.
- Repo is **private**; flip to public + GitHub Pages if a shareable live URL is wanted.

---

## Rebuild

```sh
python3 build_states.py     # regenerates index.html
python3 build_mapper.py      # regenerates segment-mapper.html
```

No build step beyond Python 3 (stdlib only). Open the HTML directly or serve the folder.
