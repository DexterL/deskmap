# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A pure browser-side seat planning tool — single HTML file, zero dependencies. Open `deskmap.html` directly in a browser, no build/dev server needed. Full spec at `docs/spec.md`, data schema at `docs/data-schema.md`.

## How to preview changes

```bash
open deskmap.html
```

There is no build step, linter, or test suite.

## Architecture

Everything lives in `deskmap.html` (HTML structure + embedded CSS + embedded JS in an IIFE).

**Layout**: Flexbox-based three-region layout — left panel (people list + staging area), toolbar, and workspace grid.

**State model** — single `state` object holds all data:

```js
state = {
  version: 2,                               // schema version
  groups: [{ id, name, color }],            // groups define person colors
  people: [{ id, name, groupId }],          // all people linked to groups
  seats: [{ id, x, y, orientation, personId | null }],
  obstacles: [{ id, x, y, w, h, name }],
  activeGroupId: string,
}
```

Assignment is tracked via `seat.personId` — a person is "in waiting list" when no seat references their ID. No separate assignment table.

**Rendering**: `renderAll()` is the central re-render entry point, calling `renderPeopleList()` and `renderWorkspace()`. Workspace items are absolutely-positioned DOM elements inside the scrollable `#grid-area`. The grid is pure CSS (`background-image` with `linear-gradient`).

**Drag system**: Pointer Events API (not HTML5 Drag API). State machine: IDLE → pointerdown → DRAGGING → pointerup/Escape → commit/cancel → IDLE. Global `drag` variable holds transient drag state (`{ type, data, ghostEl, sourceSeatId }`). `document`-level `pointermove`/`pointerup` listeners drive all drag operations. Ghost element (`#drag-ghost`) is `position: fixed` with `pointer-events: none`.

Drag types: `person`, `assigned-person`, `seat-template`, `object-move`, `obstacle-template`, `custom-obstacle-create`, `batch-move`.

**Key utility functions**: `rectsOverlap(a, b)` — AABB collision where shared edges don't count. `clientToGrid(e)` — snaps pointer coords to 30px grid via `Math.round`. `escHtml(s)` — DOM-based XSS guard for user-provided names. `getContrastColor(hex)` — returns `#333` or `#fff` based on background luminance. `validateState(data)` — validates schema version, required fields, and field types on import.

**Grid**: 30px cells. All coordinates are multiples of 30. Snapping via `Math.round(raw / 30) * 30`. Seat = 3×2 cells (90×60px). Obstacle sizes defined in cells, rendered in px.

**Persistence**: `localStorage` key `seats-tool-state`. Auto-saved on every mutation. Export/import via JSON file download/upload. Schema versioning (`SCHEMA_VERSION = 1`) ensures data compatibility.

**People list**: No hardcoded presets — starts empty. Populated via:
1. JSON import (seats + people + obstacles)
2. Single-add input with color picker
3. Batch-add modal (paste names separated by comma/newline, with starting color selection)
A "清空" button removes all people (and their seat assignments).

**Color picker**: 20-color palette displayed as clickable dots. Used in:
- Add person form (freely pick any color, click again to deselect to auto-cycle)
- Batch-add modal (pick starting color for cycling)
- Person card modification (click color dot to open popup with only unused colors)

## Key design decisions

- **Seat visual design** — CSS-only chair appearance: gradient backrest, rounded cushion, bottom corners more rounded (14px) to indicate the "front" direction. No arrow indicator needed — shape itself expresses orientation.
- **Seat rotation in workspace** — press `R` to rotate clockwise (90° steps) while hovering or dragging. Collision detection prevents rotation if it overlaps existing items.
- **Orientation is set on drag-from-staging** — the staging area has 4 directional seat templates (Right, Down, Left, Up). These map to 0, 90, 180, 270 degrees respectively.
- **Multi-Selection** — click and drag on the workspace background to draw a selection marquee. Selected items can be moved together (Batch Move).
- **Inward Selection Border** — selection outlines are inset by 3px to avoid visual merging of adjacent selected items.
- **Custom obstacle sizing** — a "自定义尺寸" toggle button enters a mode where clicking and dragging on the workspace defines a custom-sized obstacle (minimum 1×1 cell).
- **Data schema versioning** — all save/export includes a `version` field. `validateState()` checks version compatibility and field types. Old data (no version) is accepted; future incompatible versions are rejected with clear error messages.
- **Deleting a seat** with a person silently returns the person to the waiting list (person stays in `state.people`, just no longer referenced by any seat).
- **Swapping**: dragging an assigned person onto another occupied seat swaps both. Dragging an unassigned person onto an occupied seat takes the seat and displaces the occupant to the waiting list.
- **Collision detection** uses AABB overlap — shared edges do NOT count as collision, allowing adjacent placement.
- **Right-click direct delete** on seats/obstacles. Instantly removes the object and returns assigned person to list.
- **Edge auto-scroll** — when dragging near the workspace edge, the viewport scrolls automatically.
- **Color system** — 20-color palette cycled via `nextColorIdx`. Each person gets a unique color from this cycle.
- **Obstacle rename** — double-clicking an obstacle creates a temporary `<input>`; blur, Enter, or Escape finalizes/cancels the edit.
- **Clear workspace** — toolbar button removes all seats and obstacles but preserves the people list and groups.
- **Personnel Management** — supports group filtering and batch addition. Group colors are globally synced.
