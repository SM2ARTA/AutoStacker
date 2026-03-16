# AutoStacker — Pallet Stack Builder

## Project Overview
Interactive isometric pallet stacking visualizer. Define box types (W×D×H in inches), place them on a pallet, and visualize the stack in real-time with an isometric 3D canvas, top-down height map, and SVG blueprint export.

## Architecture
- **Single file**: Everything lives in `index.html` (~2,100 lines). No build step, no bundler.
- **Backend**: Supabase for save/load (`shared_state` table, key-value storage)
- **External libs**: Google Fonts only (Barlow, Barlow Condensed, JetBrains Mono). No JS dependencies.
- **Rendering**: Two `<canvas>` elements — isometric 3D view (`#isoC`) and top-down height map (`#topC`), both redrawn on every state change via `render()`.
- **Supabase**: `https://stwopndhnxcjyomkufii.supabase.co` — public anon key in source
- **Git**: remote=https://github.com/SM2ARTA/AutoStacker.git

## Layout (3-panel, viewport-locked)
```
┌─────────────────────────────────────────────────┐
│ .hdr  — brand + mode toggle + actions           │
├──────┬────────────────────────────┬──────────────┤
│ .lp  │ .ca                       │ .rp          │
│238px │ flex:1                     │ 205px        │
│      │ ┌────────────────────────┐ │ Top-down     │
│Pallet│ │ #isoC (iso canvas)     │ │ height map   │
│ size │ │ + mode overlay         │ │ #topC        │
│      │ │ + hint bar             │ │              │
│New   │ └────────────────────────┘ │ Legend       │
│ box  │ .tb — stats + explode     │              │
│      │ sliders + keyboard hints  │              │
│Auto  │                           │              │
│Build │                           │              │
│Types │                           │              │
│ list │                           │              │
└──────┴────────────────────────────┴──────────────┘
```
- `html,body { height:100%; overflow:hidden }` — no page scroll
- `.main { display:flex; flex:1; overflow:hidden }`

## Global State (`S`)
| Field | Type | Description |
|---|---|---|
| `palletW`, `palletD` | int | Pallet dimensions in inches (default 48×40) |
| `boxes[]` | array | Placed boxes: `{id, gx, gy, gz, w, d, h, color, sku, name}` |
| `past[]`, `future[]` | array | Undo/redo stacks (JSON-stringified `boxes` snapshots, max 80) |
| `types[]` | array | Box type defs: `{id, sku, name, w, d, h, color, orient, weight, flat}` |
| `activeId` | string | Currently selected box type ID |
| `hGx`, `hGy` | int\|null | Hover grid position (for preview ghost) |
| `zoom` | float | Canvas zoom level (0.15–5.0) |
| `panX`, `panY` | float | Canvas pan offset in pixels |
| `mode` | `'place'\|'select'\|'move'` | Current interaction mode |
| `selId` | int\|null | Picked-up/selected box ID |
| `viewRot` | int | View rotation 0–3 (×90°) |
| `viewMode` | `'iso'\|'top'` | Isometric or top-down orthographic |
| `explode` | float | 0–1 vertical layer separation |
| `explodeH` | float | 0–1 horizontal box separation |
| `nxtId` | int | Auto-increment ID counter for new boxes |

## Isometric Projection
- **Grid unit = 1 inch**. Constants: `BCW=5.0` (px/inch X), `BCH=2.5` (px/inch Y), `BZH=7.0` (px/inch Z)
- `toS(gx, gy, gz)` — grid → screen coordinates (standard isometric: X goes right-down, Y goes left-down, Z goes up)
- `fromS(sx, sy)` — screen → grid (inverse, assumes gz=0 ground plane)
- Origin at `(LW/2 + panX, LH*0.52 + panY)`
- `_vrot(gx,gy)` / `_vunrot(gx,gy)` — view rotation around pallet center
- Box rendering: 4 side faces + top, shading varies by `viewRot`
- Draw order: topological sort via `isoSort()` (Kahn's algorithm) for correct occlusion

## Explode View
- **Vertical** (`S.explode`, slider `#explodeSlider`): layers separate vertically, 16" gap at full
- **Horizontal** (`S.explodeH`, slider `#explodeHSlider`): boxes push outward from pallet center
- **Play button** (`#explodePlayBtn`): animates both sliders smoothly, reverses direction on completion
- Explode is purely visual — does not affect actual box positions, undo, or save/load
- Ghost previews (place/move mode) also offset when exploded

## Box Type Orientation System
- Raw dimensions stored as `t.w`, `t.d`, `t.h`
- `t.orient` (0–5) selects a permutation from `_PERMS` array
- `_PERMS = [[0,1,2],[1,0,2],[0,2,1],[2,0,1],[1,2,0],[2,1,0]]`
- `tw(t)`, `td(t)`, `th(t)` return displayed W/D/H after permutation
- `t.flat` constrains to orient 0–1 only (no vertical flips)

## Inline Editing
- **Dimensions**: click any W/D/H value in the type list → inline input, commits on blur/Enter, Esc cancels. Updates placed boxes of same type.
- **Weight**: inline number input per type
- **Flat**: inline checkbox per type
- **Color**: invisible color picker overlaid on the swatch. Changes propagate to placed boxes.

## Key Functions
| Function | Purpose |
|---|---|
| `render()` | Master render — calls `renderIso()` or `renderTopView()`, `renderTop()`, `updStats()` |
| `renderIso()` | Clears and redraws iso canvas with pallet, explode offsets, ghosts, and all boxes |
| `renderTopView()` | Orthographic top-down placement view |
| `renderTop()` | Redraws the top-down height map canvas |
| `drawBox(gx,gy,gz,w,d,h,color,alpha,sel)` | Draws one isometric box (4 faces + top + label + selection) |
| `drawPallet()` | Draws the pallet base with grid lines, rulers, crosshairs |
| `isoSort(boxes)` | Topological sort for correct isometric draw order |
| `gravZ(gx,gy,w,d,excl)` | Gravity: finds highest Z at a grid region (for stacking) |
| `inBounds(gx,gy,w,d)` | Checks if box fits within pallet bounds |
| `placeBox(gx,gy)` | Places active type at grid position (with gravity) |
| `removeTop(gx,gy)` | Removes topmost box at grid position (right-click) |
| `flipType(id)` | Cycles through valid orientations for a box type |
| `renderBTL()` | Re-renders the box type list in the left panel |
| `autoBuild()` | Auto-fills all types with qty>0 using layer packing |
| `autoFillType(id)` | Auto-fills one type using layer packing (▶ button per type) |
| `exportSVG()` | Exports exploded layer-by-layer SVG blueprint with orthographic views |
| `_psbSnapshot()` / `_psbRestore(snap)` | Save/load state to/from JSON |
| `showSaveModal()` / `showLoadModal()` | Supabase save/load UI |

## Interaction Modes
### Place Mode (default)
- **Left-click**: Place selected box type at grid position (auto-stacks via gravity)
- **Right-click**: Remove topmost box at position
- **Hover**: Ghost preview (green outline if valid, red ✕ if out of bounds)

### Select Mode
- **Left-click**: Toggle selection of topmost box at position
- **Delete/Backspace**: Remove selected box
- **Hover**: Highlight

### Move Mode
- **Left-click (no selection)**: Pick up topmost box at position
- **Left-click (with selection)**: Drop box at new position (re-stacks via gravity)
- **Right-click / Esc**: Cancel pick-up

### Canvas Navigation
- **Scroll wheel**: Zoom in/out (×1.1 / ×0.91 per tick)
- **Middle-mouse drag**: Pan the view

## Keyboard Shortcuts
| Key | Action |
|---|---|
| `F` | Flip/rotate active box type orientation |
| `M` | Cycle modes: Place → Select → Move |
| `T` | Toggle Isometric / Top-Down view |
| `L` | Duplicate top layer |
| `R` | Remove top layer |
| `Q` / `E` | Rotate view ±90° |
| `Delete` | Remove selected box or top box at hover |
| `Esc` | Cancel selection / exit move mode |
| `Ctrl+Z` | Undo |
| `Ctrl+Y` / `Ctrl+Shift+Z` | Redo |

## Design Tokens (`:root`)
| Token | Value | Usage |
|---|---|---|
| `--bg` | `#0d0e0e` | Page background (near-black) |
| `--surf` | `#141616` | Panel/header surface |
| `--surf2` | `#1a1c1c` | Input/card background |
| `--surf3` | `#1f2121` | Hover/toast background |
| `--bdr` | `#242626` | Primary border |
| `--bdr2` | `#2c2e2e` | Secondary border |
| `--acc` | `#F5A623` | Accent / primary amber |
| `--acc2` | `#FFD166` | Accent light (flip indicator, flipped dims) |
| `--acc-dim` | `rgba(245,166,35,0.12)` | Accent surface |
| `--blue` | `#5B8DEF` | Move/select mode, selection highlight |
| `--blue-dim` | `rgba(91,141,239,0.12)` | Blue surface |
| `--txt` | `#ddd8ce` | Text primary |
| `--muted` | `#6a6560` | Text secondary/muted |
| `--red` | `#e05858` | Danger / delete |
| `--green` | `#5CC87A` | Success |

### Fonts
- **UI**: `Barlow` (400/500/600)
- **Headers/buttons**: `Barlow Condensed` (600/700/900), uppercase, letter-spacing
- **Data/mono**: `JetBrains Mono` (400/500)

### Button Families
Four button families — never mix across contexts:

**`.btn`** — header bar and panel action buttons
`border:none; border-radius:2px; font-family:Barlow Condensed; font-weight:700; font-size:0.78rem; letter-spacing:1.5px; text-transform:uppercase; padding:6px 13px`
Variants:
- `.btn-acc` — amber accent fill (primary action: Add, SVG export)
- `.btn-ghost` — transparent + border (secondary: Undo, Redo, Save, Open, View toggle)
- `.btn-danger` — transparent + red text (destructive: Clear)
- `.btn-mode` — mode toggle base (`.on-place` = amber, `.on-move` = blue, `.on-select` = purple)
Modifiers:
- `.btn-sm` — `padding:5px 10px; font-size:0.72rem`
- `.btn-full` — `width:100%; justify-content:center`
- `:disabled` — `opacity:.35; cursor:not-allowed`

**`.preset-btn`** — pallet size presets only
`background:var(--surf2); border:1px solid var(--bdr); border-radius:4px; padding:4px 10px; font-size:0.62rem; font-family:Barlow Condensed; font-weight:600; letter-spacing:0.8px; color:var(--muted)`
Hover: amber border + text

**`.bti-btn`** — box type list item actions (inline, tiny)
`background:none; border:1px solid transparent; color:var(--muted); font-size:0.8rem; padding:2px 5px; border-radius:2px`
Sub-variants via class:
- `.flip-btn:hover` — amber accent
- `.del-btn:hover` — red
- `[data-autofill]` — blue (`color:var(--blue)`)

**`.modal-btn`** — modal dialog actions only
`border:none; border-radius:4px; font-family:Barlow Condensed; font-weight:700; font-size:0.75rem; letter-spacing:1.2px; text-transform:uppercase; padding:8px 18px`
- `.modal-btn-primary` — amber accent fill
- `.modal-btn-ghost` — transparent + border

**`.flip-inline`** — single-use: status bar flip button
`background:none; border:none; color:var(--acc2); font-size:0.95rem`

**`.modal-item-del`** — single-use: delete button in load modal list items
`background:none; border:none; color:var(--muted); font-size:0.85rem; padding:2px 6px`

### Inline Editable Dimension Spans
- `.dim-v` — clickable dimension value in type list (hover: amber bg + white text)
- `.dim-ed` — inline edit input replacing `.dim-v` on click (38px wide, monospace, amber border)

## Key Element IDs
| ID | Element |
|---|---|
| `isoC` | Isometric canvas |
| `topC` | Top-down height map canvas |
| `cvWrap` | Canvas wrapper (resize observer target) |
| `modeBtn` | Place/Select/Move toggle |
| `viewBtn` | Iso/Top-Down toggle |
| `undoBtn`, `redoBtn` | Undo/Redo buttons |
| `clearBtn` | Clear all boxes button |
| `exportBtn` | SVG export button |
| `saveBtn`, `loadBtn` | Save/Open buttons |
| `dupLayerBtn`, `rmLayerBtn` | Layer operations |
| `explodeSlider` | Vertical explode range input |
| `explodeHSlider` | Horizontal explode range input |
| `explodePlayBtn` | Explode animation play/pause |
| `btl` | Box type list container |
| `typeCnt` | Box type count badge |
| `pw`, `pd` | Pallet width/depth inputs |
| `applyP` | Apply pallet size button |
| `nSku`, `nName`, `nW`, `nD`, `nH`, `nClr`, `nWt`, `nFlat` | New box type form inputs |
| `addTypeBtn` | Add box type button |
| `abMaxH`, `abMaxWt`, `abOverhang`, `abInterlock` | Auto build settings |
| `sBoxes`, `sHeight`, `sFill`, `sWeight` | Stats display spans |
| `sBreakdown` | Per-type box count breakdown |
| `aPill`, `aPillTxt` | Active type pill in toolbar |
| `flipBtn` | Inline flip button in toolbar |
| `modeOverlay`, `overlayTxt` | Mode overlay banner |
| `cvHint` | Canvas hint bar |
| `toast` | Toast notification |

## Supabase Persistence
- **Save/Load**: `_psbSnapshot()` captures full state → upserts to `shared_state` table
- **Index key**: `psb-index` — list of `{id, name, date}` entries
- **Pallet key**: `psb-{timestamp}` — full JSON snapshot
- **Snapshot fields**: `palletW`, `palletD`, `types[]`, `qtys{}`, `activeId`, `boxes[]`
- Explode/viewRot/zoom are transient — not saved

## SVG Export
- Exploded layer-by-layer isometric view with `EXP_GAP=12"` between layers
- Includes pallet base, all boxes in proper draw order
- Blueprint grid with orthographic side/top/front views
- Layer instruction table with box inventory
- Fixed projection (independent of canvas zoom/pan)
- Download as `pallet-{timestamp}.svg`

## Auto Build
- **Settings**: Max Height, Max Weight, Overhang toggle (50% off edge), Interlock toggle (alternate layer rotation)
- **Per-type ▶ button**: fills pallet with just that type
- **`autoBuild()`**: fills all types with qty>0 sequentially
- Algorithm: tries all 6 orientations, finds best footprint, generates row/column layouts, stacks to max height/weight

## Dev Environment Notes
- **Shell**: Claude Code uses `C:\Users\vla8529\PortableGit-new\usr\bin\bash.exe`
- **Fork limitation**: msys2 programs (ls, grep, etc.) fail to fork under Node.js. Use Read/Grep/Glob tools instead.
- **Windows executables work fine** in Bash (e.g. `git.exe`)
- **No admin rights** on this machine (FIFA corporate domain)
- **Git**: user.name=SM2ARTA, user.email=sm2arta@outlook.com

## Editing
- Single file ~2,100 lines — use `offset` + `limit` when reading sections
- Use `Grep` with line numbers to locate functions before editing
- Prefer `Edit` over full rewrites
- All rendering flows through `render()` — any state change should call it
- Undo only covers `S.boxes` mutations — type list changes and pallet resize are not undoable
