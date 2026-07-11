# LTD Traverse Calculator — Project Context

A single-file HTML/JS PWA-style field surveying calculator, built for use on Samsung
Android work phones (installed via "Add to Home Screen" in Chrome, runs offline).

Currently `index.html` is one self-contained file: inline `<style>`, inline `<script>`,
no build step, no external dependencies except system fonts. This was intentional for
fast iteration and trivial deployment (just open the file / host it anywhere). Keep this
constraint in mind — don't introduce a build step or external packages without discussing
it first, since the deployment model (copy one file to a phone) depends on it.

## Tech / conventions

- Vanilla JS, no framework. Global `state` object + `render()` re-renders the active tab's
  `innerHTML` from scratch on every state change. No virtual DOM — this is intentional
  for simplicity, but means event listeners on dynamically-created inputs must be
  re-attached after every render (see `onBearingInput`, `onBacksightInput`, etc.)
- Bearings are **whole-circle azimuths** (0–360°, no quadrant/N-S-E-W letters), entered
  in `DDD.MMSS` field format (e.g. `125.3045` → 125°30'45"). See `parseDDDMMSS()` and
  `isBearingComplete()`. Trailing digits are optional and pad-right with zero
  (`45.3` → 45°30'00"), not left — this matches how field data collectors are typically
  used.
- Coordinates are Northing/Easting (`n`, `e`), not X/Y — surveying convention.
- `dist * cos(az)` = ΔNorthing ("lat" in the code, legacy compass-rule terminology),
  `dist * sin(az)` = ΔEasting ("dep"). This naming (`lat`/`dep`) is used internally in
  the math functions but UI labels always say "ΔNorthing"/"ΔEasting" — surveyor-facing
  language should stay in Northing/Easting, not the old lat/dep jargon.
- Design system: dark navy-black background, Parker Scanlon brand palette — navy
  (`#1B3B6F`) and gold/yellow (`#FFC220`) as the primary accents, a bright aluminum
  silver (`#E4E7EA`) as the secondary/data accent. All colors are CSS custom properties on
  `:root` (`--accent`, `--accent-2`, `--navy`, etc.) — inline SVGs (compass, plot)
  reference the same `var(--x)` values rather than duplicating hex codes, so the
  palette only needs to change in one place. A 3px navy→yellow→silver gradient strip
  (`.brand-strip`) at the very top of the app is the one deliberately "branded" touch;
  everything else stays flat and functional. Monospace (`JetBrains Mono`) for all
  numeric/coordinate data, Space Grotesk for headers/buttons. Bottom tab nav,
  mobile-first, big tap targets.

## Current features (all working, tested)

**Traverse tab**
- Add/edit/delete legs (bearing `DDD.MMSS` + distance). Editing loads a leg back into the
  entry form (pencil icon), highlights the row being edited, Update/Cancel buttons.
- Bearing input auto-advances focus to the Distance field once 4 fractional digits are typed.
- Live compass needle SVG updates as you type a bearing.
- Optional "Backsight Orientation Check" panel (collapsible, off by default): Known vs
  Observed bearing at a Starting backsight and/or Finishing backsight. Shows the signed
  difference as a genuine independent angular check (see note below on why this matters).
- Live "Closing Leg" panel: bearing + distance needed to close the loop back to the start,
  recalculated on every render (i.e. every add/edit/delete).

**Closure tab**
- Summary stats, in this order: Linear Misclose, Misclose Bearing, Perimeter, Precision Ratio.
  - **Misclose Bearing** (not "angular misclose") is the direction of the closing-error
    vector (`atan2(ΣΔE, ΣΔN)`). This exists because a true angular misclosure CANNOT be
    derived from independently-entered bearings — the turned-angle sum is a telescoping
    identity that cancels regardless of any bearing error (verified numerically during
    development, see conversation history if resurrecting this). Misclose Bearing is the
    honest substitute: it tells you the *direction* of the error so you can compare it
    against individual leg bearings to spot which leg is the likely blunder.
  - Genuine angular error detection instead lives in the Traverse tab's Backsight check,
    which compares an independently-known control bearing against what was actually
    observed — that's real, this derived-from-bearings approach is not.
- ΔEasting/ΔNorthing per-leg table with a Σ sum row.
- Traverse Adjustment: Compass (Bowditch) or Transit rule toggle. Table shows Measured vs
  Computed bearing/distance per leg (computed = derived from the adjusted lat/dep, not the
  correction values directly — more intuitive for field use than showing raw corrections).
- Enclosed Area (m² and hectares), via `computeArea()` — shoelace formula over the
  *adjusted* (closed) traverse polygon, built from relative coordinates starting at
  (0,0) since area is translation-invariant and shouldn't depend on the Coordinates
  tab's starting N/E. Shown as its own stat-grid below the main Closure Summary grid
  (kept separate rather than appended to that 4-stat grid, to avoid disturbing its
  documented Linear Misclose / Misclose Bearing / Perimeter / Precision Ratio order).

**Coordinates / Plot tab**
- User-set starting N/E. Full coordinate table + SVG grid-paper-style plot of the adjusted
  traverse loop.

**Calculator tab** (formerly "Missing Line") — 5 modes: a 2×2 toggle grid plus one
full-width button below it for the most recently added tool:
1. Inverse (2 Pts) — bearing/distance between two known N/E coordinates.
2. Closing Leg — same calc as the live Traverse tab panel, shown standalone here too.
3. Radiations — bearing/distance between two points each defined by bearing+distance
   ("radiated") from the same occupied station.
4. Bearing Compare — signed angle + turn direction (CW/CCW) between two bearings.
5. Offset / Chainage — a stakeout tool: given a baseline (points A→B), a chainage
   along it, and a left/right offset from it, computes the target point's N/E. See
   `computeOffset()` — the target is A plus `chainage` along the baseline's unit
   vector plus the signed offset along its right-perpendicular unit vector
   `(-dE/L, dN/L)` (derived from, and consistent with, the sign convention verified
   during development: facing baseline azimuth θ, "right" is azimuth θ+90°). Each
   baseline endpoint (A and B) has a `<select>` (`off-a-point`/`off-b-point`,
   populated by `offsetPointOptions()` from `computeCoordinates()`) letting you pick
   an existing traverse point instead of retyping its N/E — picking a point fills
   and read-only-locks the N/E inputs (`onOffsetPointSelect()`); "Custom" unlocks them
   for manual entry. This was originally the reverse calc (P → chainage/offset); it
   was changed to this stakeout direction and the old mode was dropped, not kept as
   a toggle, since that's what was asked for — reintroduce it as a toggle if the
   locate-a-point-on-a-baseline direction turns out to be needed too.
   Internal identifiers still say `calcMode`/`renderCalculator`/`setCalcMode` (renamed
   from the old `missingMode`/`renderMissing`/`setMissingMode` together with the UI
   label) — keep code and UI names in sync here, unlike the intentional lat/dep vs
   Northing/Easting split elsewhere in this file.

## Known limitations / likely next steps

- **Persistence (localStorage) is now implemented.** State auto-saves to `localStorage`
  (key `traverseFieldBook.v1` — kept as-is despite the app's later rename to "LTD
  Traverse Calculator", so any already-saved field job isn't silently orphaned under
  a key nothing looks for anymore) on every data-mutating action — add/edit/delete leg,
  method toggle, start coordinate change, backsight fields — via `saveState()`, and
  restores on page load via `loadState()` (both near the top of the script, right after
  the `state` object). Tab switches and other pure-navigation renders do NOT trigger a
  save, so `saveState()` is called explicitly from each mutating function rather than
  hooked into `render()` globally — keep that pattern if you add new state-mutating
  actions. The header's "NEW JOB" button (`resetJob()`) wipes it with a `confirm()`
  prompt, to a genuinely empty job (no legs) — it does NOT reload the sample job below.
  Still missing: JSON/CSV export-import so a job can be backed up or moved between
  devices, and IndexedDB if job size ever outgrows localStorage's ~5MB limit (unlikely
  for a single traverse, but worth knowing).
- **First-ever load seeds a sample job.** If `loadState()` finds no saved data at all
  (fresh install, or storage was cleared outside the app), it calls `loadSampleJob()` —
  a small closed 4-leg traverse (~0.8 ha, closes to within 0.4mm, so the Closure tab
  shows a realistic "CLOSED" badge rather than an empty state) — and saves it, so the
  app is never a blank slate the very first time someone opens it. This is distinct
  from `resetJob()`, which always clears to genuinely empty. If you add more sample
  data (e.g. to demo backsight or the Calculator tools), extend `loadSampleJob()`.
- **No true PWA manifest/service worker yet** — currently works via "Add to Home Screen"
  in Chrome but isn't a real installable/offline-cached PWA. Needs `manifest.json` +
  a service worker for real offline reliability.
- The Backsight Orientation Check (Traverse tab) is currently self-contained and does NOT
  feed into the Closure tab's stats. Wiring the finishing-backsight misclosure into the
  Closure summary was discussed as a possible next step but not yet built.
- Calculator tab tools are stateless scratch pads (values aren't saved to `state`,
  just read from the DOM on "Calculate" click) — fine for a quick calculator, but worth
  knowing if you're looking for where that data "lives." `state.calcMode` (which
  sub-tool is active) also isn't persisted to localStorage, consistent with this.
- No multi-job / multi-file support — one traverse per session.
