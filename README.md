# CNC Program Viewer

A single-file, browser-based viewer for CNC toolpath programs. Drop in a file and it parses, renders, and lets you step through the program — no install, no server, nothing leaves your browser.

**Live version:** open [`gcode-viewer.html`](./gcode-viewer.html) directly, or use the hosted copy if you've set one up (e.g. via GitHub Pages). A **Help** button in the header opens this same overview as a pop-up card inside the tool.

## Supported formats

| Format | Typical extensions | Notes |
|---|---|---|
| Fanuc-style G-code | `.anc`, `.nc`, `.tap`, `.gcode`, `.txt`, `.ncf`, `.cnc` | G0/G1/G2/G3, R- and I/J-based arcs, G41/G42 cutter compensation, tool changes |
| Biesse / Maestro | `.xcs` | Scripted CAM format (`CreatePolyline`, `CreateRoughFinish`, `CreateDrill`, `CreatePart`, etc.) |
| SCM Xilog+ | `.xxl` | SCM router post format |
| ISO-ESAGV | `.i00` | Same G-code vocabulary as Fanuc, but with absolute (not relative) I/J arc centers |

The format is auto-detected from file contents — you don't need to pick it manually.

## Getting started

1. Open `gcode-viewer.html` in any modern browser (double-click it, or host it somewhere and open the URL).
2. Drag one or more program files onto the drop zone, or click it to browse. You can load several files at once and switch between them in the left-hand file list.
3. The first successfully-parsed file is opened automatically.

## Layout

- **Files panel (far left)** — every loaded file, with a size/format badge. Files with audit problems show a warning icon (see [Corner & compensation audit](#corner--compensation-audit-gcode--iso-esagv) below) — hover it for a summary. Drag a file to reorder the list, or click its **×** to remove it.
- **Info panel** — job/panel details (material, panel size, origin), program stats (counts, ranges), and either a **Tool changes** list (G-code/Xilog/ESAGV) or a **Parts** list (xcs).
- **Viewer** — the toolpath itself, plus a synced code panel showing the raw file.

## Viewing the toolpath

- **Fit to view** re-centers and re-scales the drawing.
- **Scrub slider + Play** animate the program move-by-move; the code panel highlights the current line and scrolls to follow.
- Click anywhere in the **code panel** to jump the viewer to that line.
- **Code panel: Bottom / Right** toggles whether the code panel docks under or beside the canvas; drag the divider between them to resize.
- **Show rapids** toggles whether rapid (non-cutting) travel moves are drawn.
- Mouse wheel to zoom, click-drag to pan.

### Searching the code panel

The search bar docked above the code panel highlights every line containing your text as you type (case-insensitive). Use the ▲/▼ buttons — or `Enter` / `Shift+Enter` right from the input — to step to the next or previous match, with a counter showing your position (e.g. `3 / 12`). `Escape` or the **×** button clears the search.

### Parts list (xcs files)

For Maestro `.xcs` files, the secondary panel lists every part on the sheet instead of tool changes:

- Click a part to highlight it, zoom the viewer to just that part, and jump the code panel to that part's line; click again to collapse.
- **◀ Part / Part ▶** buttons in the toolbar step through parts one at a time (wraps around at either end) — handy for reviewing a whole nest without hunting through the list.
- Expand a part to see its individual operations (drills, routing passes) and jump straight to the line that produces each one.
- **Stacked-part alert:** if two parts occupy essentially the same footprint (≥95% of each other's bounding-box area) — almost always a duplicate or mis-nested part rather than normal edge-to-edge nesting — both are highlighted in red with a warning icon. Hover the icon to see which part(s) it overlaps.

## Corner & compensation audit (G-code / ISO-ESAGV)

For Fanuc-style and ISO-ESAGV files, the info panel includes an audit of geometry that's known to break cutter-radius compensation (G41/G42) mid-cut:

- **Colinear runs** — a string of points that are effectively one straight line, split across multiple G-code blocks. These collapse safely into a single segment with no path change, and are marked **auto-fix**.
- **Short-sharp segments** — a segment shorter than the currently-active tool radius sitting at a real corner. These can't be resolved automatically and are marked **review** for you to inspect manually.

Each finding is its own card. You can:

- **Apply this fix** on an individual card, or check several and use **Apply selected** (there's also a **Check all** button).
- Once at least one fix has been applied, a **Download fixed file** button appears — it downloads a new file with only the confirmed lines removed. The originally-loaded file is never modified; applied fixes are shown as struck-through lines in the code panel so you can compare before/after.

### Resume export

Also under the audit section: pick a tool change and an operation from the dropdowns, then **Export from selected operation** to download a version of the file that replays that tool's setup (tool change, spindle start, length offset) and then jumps straight to the chosen operation — for restarting a job partway through a sheet without re-running everything from the top. Nothing is recalculated; only existing lines are kept or skipped. Always review the exported file before running it on a machine.

## Nesting-label export (xcs only)

Maestro files place a `SetNestingPartLabel(...)` call per part to auto-print a label. If you need the controller to skip that (e.g. pre-printed labels, or a different labeling workflow), the **Nesting labels** section provides a one-click **Comment out N label lines & download** button. It prefixes every `SetNestingPartLabel` line with `//` and downloads a new file — again, without touching the loaded original.

## Depth, tabs & onion-skin (G-code / xcs)

A second audit, separate from the corner/compensation one above, checks how deep every toolpath actually cuts and flags a couple of nest-design patterns worth a second look. It applies to Fanuc-style G-code and Maestro `.xcs` files.

- **Depth audit** — on Onsrud/Fanuc `.anc` files, the spoilboard surface is Z0; up to 0.1mm below that is normal blow-through and isn't flagged. Between 0.1mm and 0.25mm past Z0 gets a yellow **DEPTH** review flag; 0.25mm or deeper gets a red **DEPTH ⚠** hard-limit flag. On Biesse X200 `.xcs` files the same two tiers apply, measured against blank thickness + tolerance instead of Z0, for both routing passes and drill depths.
- **Tabs** — on xcs this comes directly from the file's own `SetAttribute("TAB", ...)` calls; on G-code it's inferred from the path lifting a few mm for roughly 15–90mm of travel before dropping back down (a normal full-height retract isn't flagged). A file with any tabs gets a **TABS** tag next to its name in the file list. On the canvas, each G-code tab draws as a solid red line over its exact lift/travel/drop-back bump — but only once the scrub/play animation has actually cut through it, same as the rest of the toolpath. Hover the red line for a tooltip with bottom Z, top Z, and tab length in mm and inches.
- **Onion-skin (xcs only)** — a 🧅 appears where the same path is routed twice: once shallow (above the blank) and once as a full thru-cut. If either pass also carries a tab, a **TABS** pill is shown alongside it.

Findings live in a **Depth, tabs & onion-skin** list in the info panel — click one to jump the code panel and highlight it on the canvas. Tab pills and onion emoji are always visible on the canvas at a glance; depth issues highlight on click, since a single deep run can span many lines.

## 2"×4" label markers (xcs only)

For Maestro files with a `SetNestingPartLabel(...)` call, the viewer draws a dashed 2in×4in (landscape) rectangle at the label's position with the part's name centered inside — so you can see at a glance where each part's printed label will land on the sheet.

## A note on safety

This tool only reads and displays files, and any "fix" or "export" action produces a **new** downloaded file — it never overwrites what you loaded. That said, this is machine-facing G-code: always review a modified/exported file yourself before sending it to a controller.
