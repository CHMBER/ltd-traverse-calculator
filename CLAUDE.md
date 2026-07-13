# LTD Traverse Calculator — Project Context

A single-file HTML/JS PWA-style field surveying calculator, built for use on Samsung
Android work phones (installed via "Add to Home Screen" in Chrome, runs offline).

Currently `index.html` is one self-contained file: inline `<style>`, inline `<script>`,
no build step, no external dependencies except system fonts. This was intentional for
fast iteration and trivial deployment (just open the file / host it anywhere). Keep this
constraint in mind — don't introduce a build step or external packages without discussing
it first, since the deployment model (copy one file to a phone) depends on it.

Live at https://chmber.github.io/ltd-traverse-calculator/ via GitHub Pages (public repo
`CHMBER/ltd-traverse-calculator`, deploys from `main` root on every push — no CI config,
Pages serves `index.html` directly since there's no build step). Push to `main` to
update the live site.

## Tech / conventions

- Vanilla JS, no framework. Global `state` object + `render()` re-renders the active tab's
  `innerHTML` from scratch on every state change. No virtual DOM — this is intentional
  for simplicity, but means event listeners on dynamically-created inputs must be
  re-attached after every render (see `onBearingInput`, `onPlainBearingInput`, etc.)
- Bearings are **whole-circle azimuths** (0–360°, no quadrant/N-S-E-W letters), entered
  in `DDD.MMSS` field format (e.g. `125.3045` → 125°30'45"). See `parseDDDMMSS()` and
  `isBearingComplete()`. Trailing digits are optional and pad-right with zero
  (`45.3` → 45°30'00"), not left — this matches how field data collectors are typically
  used.
- `azimuthToBearing()` (decimal degrees → `{deg,min,sec}` for display, used everywhere
  a computed bearing is shown — Misclose Bearing, Closing Leg, Comp. Brg, Inverse/
  Radiation/Offset results) rounds once at the arcsecond level (`Math.round(az*3600)`)
  and decomposes via integer division/modulo. Previously it floored degrees, floored
  minutes, then separately rounded seconds — that cascading floor+floor+round could
  produce an invalid carry like `29'60"` instead of `30'00"` whenever the true value's
  fractional second was ≥59.5. Found and fixed during a user report of a small (~10″,
  ~1mm) discrepancy against CAD; this specific bug turned out not to explain that
  particular report (it only affects bearing *display* formatting, not `linearError`
  or any other internal math — verified the core closure formulas independently
  against hand-calculated cases, exact match to full floating-point precision), but
  was a real, confirmable bug worth fixing regardless. If you touch this function
  again, round once at the finest unit before decomposing — don't round each of
  deg/min/sec separately.
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
- Live "Closing Leg" panel at the top: bearing + distance needed to close the loop back
  to the start, recalculated on every render (i.e. every add/edit/delete). Deliberately
  first on the page — it's the number a surveyor checks most often mid-job.
- Add/edit/delete legs (bearing `DDD.MMSS` + distance) below that. Editing loads a leg
  back into the entry form (pencil icon), highlights the row being edited, Update/Cancel
  buttons.
- Bearing input auto-advances focus to the Distance field once 4 fractional digits are typed.
  The live bearing-preview line below it is blank until you start typing (no "Type
  degrees.minutesseconds" placeholder — that was judged unnecessary clutter) — this only
  applies to `#bearing-preview` specifically; the shared `bearingPreviewText()` helper
  used by Radiation/Compare still returns that hint text for their previews, unchanged.
- Distance input uses `.dist-input-lg` (20px font, same padding as `.bearing-input`) so
  it visually matches the bearing field instead of looking like a smaller, secondary
  input — they're equally primary. `.leg-entry-row` carries a 10px margin-bottom so
  there's a visible gap before the Add/Update Leg button.
- Pressing Enter (the mobile number pad's Enter/Go/Done key, via `enterkeyhint="done"`
  on `#in-dist`, or a physical Enter key) while focused in the Distance field calls
  `saveLeg()` directly (`onLegEntryKeydown()`, attached alongside the bearing input's
  listener in `render()`'s post-render hookup) — lets you finish a leg without reaching
  for the Add/Update Leg button. Works for both add and update since `saveLeg()` already
  branches on `state.editingLegId` regardless of how it's invoked.
- Live compass needle SVG (`compassSvg()`, rendered small at 76×76 — the internal
  viewBox stays `0 0 120 120` so `updateCompassNeedle()`'s coordinate math doesn't need
  to change, only the rendered size shrinks) sits beside the bearing/distance fields in
  a `.leg-entry-row` flex layout (`.leg-entry-fields` + `.compass-wrap-side`), not below
  them — keep both fields comfortably wide since they're the primary input, the compass
  is a secondary visual aid.
- There used to be an optional Backsight Orientation Check panel here (Known vs Observed
  bearing at a control point, as an independent angular check). It was removed per
  request — wasn't behaving as expected and wasn't needed. If a real angular check is
  wanted again later, don't just restore the old version verbatim without checking why
  it didn't work for the user first.

**Closure tab**
- Summary stats, in this order: Linear Misclose, Misclose Bearing, Perimeter, Precision Ratio.
  - **Misclose Bearing** (not "angular misclose") is the direction of the closing-error
    vector (`atan2(ΣΔE, ΣΔN)`). This exists because a true angular misclosure CANNOT be
    derived from independently-entered bearings — the turned-angle sum is a telescoping
    identity that cancels regardless of any bearing error (verified numerically during
    development, see conversation history if resurrecting this). Misclose Bearing is the
    honest substitute: it tells you the *direction* of the error so you can compare it
    against individual leg bearings to spot which leg is the likely blunder.
  - There is currently no genuine independent angular check in the app (a Backsight
    Orientation Check panel used to fill that role — known vs. observed bearing at a
    control point — but was removed; see Traverse tab notes above). Misclose Bearing
    above remains the only, derived-from-bearings-only substitute — keep that
    distinction in mind if this ever needs revisiting.
- ΔEasting/ΔNorthing per-leg table with a Σ sum row.
- Traverse Adjustment: Compass (Bowditch) or Least Squares toggle (`state.method` —
  `'compass' | 'leastSquares'`; the old Transit rule was dropped entirely per request,
  not kept as a third option). Table shows Measured vs Computed bearing/distance per
  leg (computed = derived from the adjusted lat/dep, not the correction values
  directly — more intuitive for field use than showing raw corrections).
  - **Least Squares** (`computeLeastSquaresCorrections()` / `legCovariance()`, both
    just above `computeAdjusted()`) is a real rigorous adjustment, not a bigger Compass
    rule: each leg gets a 2×2 (N,E) covariance matrix propagated from assumed distance
    precision (error along the leg's own direction) and assumed azimuth precision
    (error transverse to it, growing with leg length — `LS_SIGMA_DIST_CONST`/
    `_PPM`/`LS_SIGMA_AZIMUTH_ARCSEC` near the top of that block, defaulted to a modern
    total station's ~5mm+5ppm / ~5″ and easy to retune there). For a single closed
    loop the constrained-least-squares solution has a closed form — no iterative
    solver or full normal-equations matrix — via Lagrange multipliers: with per-leg
    covariance `C_i` and misclosure vector `L = (-sumLat, -sumDep)`, the correction is
    `v_i = C_i · (ΣC_i)⁻¹ · L`. Verified numerically: corrections always sum to
    exactly cancel the misclosure (to floating-point precision), and the distribution
    is genuinely different from Compass rule's pure distance-proportionality (it also
    weighs each leg's azimuth relative to the misclosure direction) — Compass rule
    falls out as the special case where every leg's covariance is isotropic and
    scales with distance, which is a decent sanity check if this ever needs revisiting.
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
   during development: facing baseline azimuth θ, "right" is azimuth θ+90°). This was
   originally the reverse calc (P → chainage/offset); it was changed to this stakeout
   direction and the old mode was dropped, not kept as a toggle, since that's what was
   asked for — reintroduce it as a toggle if the locate-a-point-on-a-baseline direction
   turns out to be needed too.
   Internal identifiers still say `calcMode`/`renderCalculator`/`setCalcMode` (renamed
   from the old `missingMode`/`renderMissing`/`setMissingMode` together with the UI
   label) — keep code and UI names in sync here, unlike the intentional lat/dep vs
   Northing/Easting split elsewhere in this file.

**Traverse point picker** — Inverse and Offset both let any N/E field be filled from
the traverse's own computed coordinates instead of retyped, via a shared trio just
above `renderOffset()`: `traversePointOptions()` (builds the `<option>` list from
`computeCoordinates()`), `pointPickerHtml(selectId, nFieldId, eFieldId, pointOptions)`
(renders the `<select>`, wired to call `onPointSelect` with those exact field IDs —
this is why it drops into any panel unchanged, no ID-naming convention required), and
`onPointSelect(selectEl, nFieldId, eFieldId)` (fills + read-only-locks the two fields,
or unlocks them again on "Custom"). If you add a point picker to another calculator,
reuse this trio rather than duplicating it — it was originally Offset-only
(`offsetPointOptions()`/`onOffsetPointSelect()`) and got generalized here specifically
so Inverse could reuse it.

## Known limitations / likely next steps

- **Persistence (localStorage) is now implemented.** State auto-saves to `localStorage`
  (key `traverseFieldBook.v1` — kept as-is despite the app's later rename to "LTD
  Traverse Calculator", so any already-saved field job isn't silently orphaned under
  a key nothing looks for anymore) on every data-mutating action — add/edit/delete leg,
  method toggle, start coordinate change — via `saveState()`, and
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
  data (e.g. to demo the Calculator tools), extend `loadSampleJob()`.
- **No true PWA manifest/service worker yet** — currently works via "Add to Home Screen"
  in Chrome but isn't a real installable/offline-cached PWA. Needs `manifest.json` +
  a service worker for real offline reliability.
- Calculator tab tools are stateless scratch pads (values aren't saved to `state`,
  just read from the DOM on "Calculate" click) — fine for a quick calculator, but worth
  knowing if you're looking for where that data "lives." `state.calcMode` (which
  sub-tool is active) also isn't persisted to localStorage, consistent with this.
- No multi-job / multi-file support — one traverse per session.
