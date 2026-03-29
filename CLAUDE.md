# Handpan Chord Diagram Generator

## Project Overview

Interactive tool for generating handpan chord diagrams for use in an e-book. Renders SVG diagrams of handpan note layouts with selectable chord highlighting. Supports multiple handpan models/scales with export to SVG and PNG.

**Owner:** Joel (Upstream Bookkeeping / handpan player)
**Primary output:** High-resolution chord diagrams for print e-book
**Instruments owned:** Ayasa E Amara 20, NEOTONE Mutant digital handpan
**First model built:** F# Pygmy 21 (Mag Instruments)

## Architecture

### Current State
Single self-contained HTML file (`index.html`) with inline JS/CSS. No build step — open in browser, select chord, export.

### Target State
- Dropdown to select handpan model (scale + note count)
- Chord library per model
- Shared rendering engine
- Still runs as a simple local HTML tool (no server needed)

### File Structure
```
handpan-diagrams/
├── CLAUDE.md              # This file
├── index.html             # Main application (current working version)
├── scales/                # One JSON file per handpan model
│   └── f-sharp-pygmy-21.json
└── reference/             # Reference images of handpan layouts
```

## Rendering Engine — How It Works

### SVG Canvas
- ViewBox: `620 × 690`
- Center point: `CX=310, CY=320`
- Background: `#f5f6f8` with `rx=12` rounded corners

### Handpan Structure (two concentric ellipses)
- **Outer shell** (bottom of handpan): `rx=278, ry=272`, fill `#d4d8df`
- **Inner shell** (top dome): `rx=178, ry=176`, fill `#c5cbd5`

### Three Rings of Notes
Notes are placed on three concentric rings using polar coordinates:

1. **Inner ring** (`IR=138`): Tone fields 1–9 orbit the Ding near the inner shell edge
2. **Middle ring** (`MR=82`): Tone fields 10–11 sit between the Ding and the top of ring 1
3. **Outer ring** (`OR=228`): Tone fields 12–20 sit centered ON the outer shell rim

### Ding Offset
The Ding (and middle ring notes 10–11) are shifted down by `DING_DROP=18` pixels from the geometric center, matching real handpan proportions where the Ding sits slightly below center on the dome.

### Clock Position System
Notes are placed using clock-hour angles converted to radians:
```js
function clock(h) { return ((h * 30 - 90) * Math.PI) / 180; }
```
- `clock(0)` = 12 o'clock (top)
- `clock(3)` = 3 o'clock (right)
- `clock(6)` = 6 o'clock (bottom)
- `clock(9)` = 9 o'clock (left)

Fractional hours give fine control: `clock(5.25)` = slightly right of bottom center.

### F# Pygmy 21 Layout (reference model)

**Inner ring (IR=138) — tone fields 1–9:**
Notes spiral from bottom, alternating right/left, ascending in pitch toward the top. Rotated CCW by 0.75 clock-hours so that E5 is centered at 12:00 and G#3/A3 straddle the bottom.

| Note   | Num | Clock Position |
|--------|-----|----------------|
| G#3    | 1   | 5.25           |
| A3     | 2   | 6.75           |
| C#4    | 3   | 3.75           |
| E4     | 4   | 8.25           |
| F#4    | 5   | 2.25           |
| G#4    | 6   | 9.50           |
| A4     | 7   | 1.00           |
| C#5    | 8   | 11.00          |
| E5     | 9   | 0.00 (12:00)   |

**Middle ring (MR=82) — tone fields 10–11:**
Shifted down by DING_DROP to stay aligned with the Ding.

| Note   | Num | Clock Position | Notes                        |
|--------|-----|----------------|------------------------------|
| F#5    | 10  | 11.00          | Between Ding and C#5         |
| G#5    | 11  | 1.00           | Between Ding and A4          |

**Outer ring (OR=228) — tone fields 12–20:**

| Note   | Num | Clock Position |
|--------|-----|----------------|
| D3     | 12  | 4.25           |
| E3     | 13  | 7.75           |
| B3     | 14  | 2.75           |
| D4     | 15  | 9.25           |
| D5     | 16  | 1.50           |
| B4     | 17  | 10.50          |
| B5     | 18  | 0.75           |
| A5     | 19  | 11.25          |
| C#6    | 20  | 0.00 (12:00)   |

### Note Sizing Rules

**Circle radius scales inversely with pitch** (lower = bigger):
- Uses MIDI-like semitone values for each note (D3=38 through C#6=73)
- Linear interpolation: `R_MAX=32` (lowest) to `R_MIN=22` (highest)
- **Ding override:** radius `44` (largest circle on the diagram)
- **Bottom ding override:** D3 and E3 get radius `38` (they are dings on the bottom shell, nearly as large as the top Ding)

```js
function getNoteRadius(note, isDing) {
  if (isDing) return 44;
  if (note === "D3" || note === "E3") return 38;
  const t = (PITCH[note] - PITCH_MIN) / (PITCH_MAX - PITCH_MIN);
  return Math.round(R_MAX - t * (R_MAX - R_MIN));
}
```

**The size range is intentionally compressed** (32 to 22, not 32 to 14) so the difference between large and small notes is subtle, not drastic.

### Font Sizing Rules
- **Note name font:** `Math.max(11.3, r * 0.42)` — floor matches F#4 size so all notes ≤ F#4 share the same readable font
- **Note number font:** `Math.max(12, r * 0.32)` — floor set to match E3's number size for consistency
- **Font family:** `Georgia, Palatino, serif`
- **Vertical layout within circle:** note name sits at `y - gap*0.5`, number at `y + gap*1.2` where `gap = r * 0.30`

### Color Palette
```js
const P = {
  shell: "#d4d8df",  shellS: "#b8bec8",   // Outer shell fill/stroke
  top: "#c5cbd5",    topS: "#a8b0bc",     // Inner shell fill/stroke
  note: "#eaecf0",   noteS: "#b0b6c0",    // Default note fill/stroke
  txt: "#5a6270",    num: "#8890a0",       // Note text / number text
  hi: "#3b7dd8",     hiGlow: "#5a9cf0",   // Highlighted note fill/glow
  hiTxt: "#fff",     hiS: "#2a5faa",      // Highlighted text/stroke
  ding: "#dde0e6",   dingS: "#a8adb8",    // Ding fill/stroke
  bg: "#f5f6f8",                           // Background
  title: "#2c3340",  sub: "#6b7585",      // Title/subtitle text
};
```

### Highlight Effect
When a note is in the selected chord:
- Fill changes to `#3b7dd8` (blue)
- An outer glow ring at `r + 4` with `#5a9cf0`, `opacity 0.45`
- Text becomes white, stroke becomes `#2a5faa`
- Font weight bumps to 700

## Adding a New Handpan Model

To add a new handpan, create a JSON file in `scales/` following this structure:

```json
{
  "name": "F# Pygmy 21",
  "manufacturer": "Mag Instruments",
  "ding": "F#3",
  "noteCount": 21,
  "notes": [
    { "note": "G#3", "num": 1, "ring": "inner", "clock": 5.25 },
    { "note": "A3",  "num": 2, "ring": "inner", "clock": 6.75 },
    ...
    { "note": "F#5", "num": 10, "ring": "middle", "clock": 11 },
    { "note": "G#5", "num": 11, "ring": "middle", "clock": 1 },
    ...
    { "note": "D3",  "num": 12, "ring": "outer", "clock": 4.25 },
    ...
  ],
  "bottomDings": ["D3", "E3"],
  "chords": {
    "F#m": ["F#3", "A3", "C#4", "F#4", "A4", "C#5", "F#5"],
    ...
  }
}
```

### Key considerations for new models:
- **Different note counts** (8, 9, 10, 17, 21, etc.) will need different ring radii and clock spacings
- **Handpans with no bottom notes** only need the inner ring (and possibly middle ring)
- **Bottom dings** vary by model — some have 0, 1, or 2 bottom dings that should be sized larger
- **The spiral pattern** (bottom → alternating left/right → top, ascending pitch) is standard across most handpans but clock positions need to be set per model
- **Reference images** should be saved in `reference/` when building a new layout

## Export

- **SVG export:** Resolution-independent vector, ideal for print
- **PNG export:** Rendered at 3× scale (1860 × 2070) for high-res print
- **Filename pattern:** `handpan-{chordname}.svg` or `.png`

## Development Notes

- No build tools needed — plain HTML/JS/CSS
- When adding the scale selector dropdown, load JSON files dynamically or bundle them inline
- The rendering engine (`buildSVG()`) should be refactored to accept a scale config object rather than using global constants
- Consider Vite if the project grows beyond ~3 files, but keep it simple for as long as possible
