# Pallet Stack Builder

## Project Overview
Interactive isometric pallet stacking visualizer. Lets users define box types (W×D×H in inches), place them on a standard pallet, and visualize the stack in real-time with an isometric 3D canvas and a top-down height map.

## Architecture
- **Single file**: Everything lives in `index.html` (~646 lines). No build step, no bundler, no backend.
- **No persistence**: All state is in-memory only. Refreshing the page resets everything.
- **External libs**: Google Fonts only (Barlow, Barlow Condensed, JetBrains Mono). No JS dependencies.
- **Rendering**: Two `<canvas>` elements — isometric 3D view (`#isoC`) and top-down height map (`#topC`), both redrawn on every state change via `render()`.

## Layout (3-panel, viewport-locked)
```
┌─────────────────────────────────────────────────┐
│ .hdr  — brand + mode toggle + undo/redo/clear/export │
├──────┬────────────────────────────┬─────────────┤
│ .lp  │ .ca                       │ .rp         │
│238px │ flex:1                     │ 205px       │
│      │ ┌────────────────────────┐ │ Top-down    │
│Pallet│ │ #isoC (iso canvas)     │ │ height map  │
│ size │ │ + mode overlay         │ │ #topC       │
│      │ │ + hint bar             │ │             │
│New   │ └────────────────────────┘ │ Legend      │
│ box  │ .tb — stats + active pill  │             │
│      │ + keyboard hints           │             │
│Types │                            │             │
│ list │                            │             │
└──────┴────────────────────────────┴─────────────┘
```
- `html,body { height:100%; overflow:hidden }` — no page scroll
- `.main { display:flex; flex:1; overflow:hidden }`

## Global State (`S`)
Single state object at line ~212:
| Field | Type | Description |
|---|---|---|
| `palletW`, `palletD` | int | Pallet dimensions in inches (default 48×40) |
| `boxes[]` | array | Placed boxes: `{id, gx, gy, gz, w, d, h, color, name}` |
| `past[]`, `future[]` | array | Undo/redo stacks (JSON-stringified `boxes` snapshots, max 80) |
| `types[]` | array | Box type definitions: `{id, name, w, d, h, color, flipped}` |
| `activeId` | string | Currently selected box type ID |
| `hGx`, `hGy` | int\|null | Hover grid position (for preview ghost) |
| `zoom` | float | Canvas zoom level (0.15–5.0) |
| `panX`, `panY` | float | Canvas pan offset in pixels |
| `mode` | `'place'\|'move'` | Current interaction mode |
| `selId` | int\|null | Picked-up box ID in move mode |
| `nxtId` | int | Auto-increment ID counter for new boxes |

## Isometric Projection
- **Grid unit = 1 inch**. Constants: `BCW=5.0` (px/inch X), `BCH=2.5` (px/inch Y), `BZH=7.0` (px/inch Z)
- `toS(gx, gy, gz)` — grid → screen coordinates (standard isometric: X goes right-down, Y goes left-down, Z goes up)
- `fromS(sx, sy)` — screen → grid (inverse, assumes gz=0 ground plane)
- Origin at `(LW/2 + panX, LH*0.52 + panY)`
- Box rendering: 3 visible faces — left (×0.68 shade), right (×0.40 shade), top (×1.14 shade)
- Painter's algorithm sort: `(gx+gy)` descending, then `gz` ascending

## Key Functions
| Function | Purpose |
|---|---|
| `render()` | Master render — calls `renderIso()`, `renderTop()`, `updStats()` |
| `renderIso()` | Clears and redraws the isometric canvas with pallet, ghosts, and all boxes |
| `renderTop()` | Redraws the top-down height map canvas |
| `drawBox(gx,gy,gz,w,d,h,color,alpha,sel)` | Draws one isometric box (3 faces + label + selection indicator) |
| `drawPallet()` | Draws the pallet base with grid lines and ruler labels |
| `gravZ(gx,gy,w,d,excl)` | Gravity: finds highest Z at a grid region (for stacking) |
| `inBounds(gx,gy,w,d)` | Checks if box fits within pallet bounds |
| `placeBox(gx,gy)` | Places active type at grid position (with gravity) |
| `removeTop(gx,gy)` | Removes topmost box at grid position (right-click) |
| `pickUp(gx,gy)` | Move mode: selects topmost box at position |
| `dropAt(gx,gy)` | Move mode: drops picked-up box at new position |
| `flipType(id)` | Swaps W↔D for a box type |
| `renderBTL()` | Re-renders the box type list in the left panel |
| `updStats()` | Updates status bar: box count, max height, fill % |
| `updModeUI()` | Updates mode button, overlay, cursor, and hint text |
| `saveHist()` / `undo()` / `redo()` | History management (JSON snapshot of `S.boxes`) |
| `toast(msg)` | Shows a temporary notification (1.9s) |

## Interaction Modes
### Place Mode (default)
- **Left-click**: Place selected box type at grid position (auto-stacks via gravity)
- **Right-click**: Remove topmost box at position
- **Hover**: Ghost preview of box placement (green outline if valid, red ✕ if out of bounds)

### Move Mode (toggle with `M` key or mode button)
- **Left-click (no selection)**: Pick up topmost box at position
- **Left-click (with selection)**: Drop box at new position (re-stacks via gravity)
- **Right-click**: Cancel pick-up
- **Esc**: Cancel pick-up, or exit move mode

### Canvas Navigation
- **Scroll wheel**: Zoom in/out (×1.1 / ×0.91 per tick)
- **Middle-mouse drag**: Pan the view

## Keyboard Shortcuts
| Key | Action |
|---|---|
| `F` | Flip active box type W↔D |
| `M` | Toggle Place/Move mode |
| `Esc` | Cancel selection / exit move mode |
| `Ctrl+Z` | Undo |
| `Ctrl+Shift+Z` / `Ctrl+Y` | Redo |

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
| `--acc2` | `#FFD166` | Accent light (flip indicator) |
| `--acc-dim` | `rgba(245,166,35,0.12)` | Accent surface |
| `--blue` | `#5B8DEF` | Move mode / selection |
| `--blue-dim` | `rgba(91,141,239,0.12)` | Blue surface |
| `--txt` | `#ddd8ce` | Text primary |
| `--muted` | `#6a6560` | Text secondary/muted |
| `--red` | `#e05858` | Danger / delete |
| `--green` | `#5CC87A` | Success (unused currently) |

### Fonts
- **UI**: `Barlow` (400/500/600)
- **Headers/buttons**: `Barlow Condensed` (600/700/900), uppercase, letter-spacing
- **Data/mono**: `JetBrains Mono` (400/500)

### Button Classes
- `.btn` — base button (Barlow Condensed, uppercase, 0.78rem)
- `.btn-acc` — amber accent (primary action)
- `.btn-ghost` — transparent with border (secondary)
- `.btn-danger` — red text (destructive)
- `.btn-sm` — smaller padding
- `.btn-full` — full width
- `.btn-mode` — mode toggle (`.on-place` = amber, `.on-move` = blue)

## Key Element IDs
| ID | Element |
|---|---|
| `isoC` | Isometric canvas |
| `topC` | Top-down height map canvas |
| `cvWrap` | Canvas wrapper (resize observer target) |
| `modeBtn` | Place/Move toggle button |
| `undoBtn`, `redoBtn` | Undo/Redo buttons |
| `clearBtn` | Clear all boxes button |
| `exportBtn` | PNG export button |
| `btl` | Box type list container |
| `typeCnt` | Box type count badge |
| `pw`, `pd` | Pallet width/depth inputs |
| `applyP` | Apply pallet size button |
| `nName`, `nW`, `nD`, `nH`, `nClr` | New box type form inputs |
| `addTypeBtn` | Add box type button |
| `sBoxes`, `sHeight`, `sFill` | Stats display spans |
| `aPill`, `aPillTxt` | Active type pill in status bar |
| `flipBtn` | Inline flip button in status bar |
| `modeOverlay`, `overlayTxt` | Move mode overlay banner |
| `cvHint` | Canvas hint bar at bottom |
| `toast` | Toast notification element |

## Default Box Types
5 preset types (furniture boxes for event logistics):
1. Chair – Flat: 24×24×18" `#C8823A`
2. Chair – Tall: 14×14×32" `#E8C060`
3. Desk Tabletop: 48×24×3" `#E2C870`
4. Desk Legs Box: 12×24×34" `#8A5618`
5. Desk Beams Box: 36×12×3" `#B89030`

## PNG Export
- Temporarily clears hover ghost and selection highlight
- Exports `#isoC` canvas via `toBlob()` → download as `pallet-{timestamp}.png`
- Restores state and re-renders after export

## Dev Environment Notes
- **Shell**: Claude Code uses `C:\Users\vla8529\PortableGit-new\usr\bin\bash.exe`
- **Fork limitation**: msys2 programs (ls, grep, etc.) fail to fork under Node.js. Use Read/Grep/Glob tools instead.
- **Windows executables work fine** in Bash (e.g. `git.exe`)
- **No admin rights** on this machine (FIFA corporate domain)
- **Git**: user.name=SM2ARTA, user.email=sm2arta@outlook.com

## Editing
- Single file — use `Grep` to locate functions before editing
- Prefer `Edit` over full rewrites
- All rendering flows through `render()` — any state change should call it
- Undo only covers `S.boxes` mutations — type list changes and pallet resize are not undoable
