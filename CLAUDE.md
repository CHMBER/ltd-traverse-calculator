# Traverse Field Book — Project Context

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
- Design system: dark graphite background, safety-orange (`#FF7A1A`) as the primary
  accent (flagging-tape reference), teal (`#4FD1C5`) as secondary/data accent (steel-tape
  reference), monospace (`JetBrains Mono`) for all numeric/coordinate data, Space Grotesk
  for headers/buttons. Bottom tab nav, mobile-first, big tap targets.

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

**Coordinates / Plot tab**
- User-set starting N/E. Full coordinate table + SVG grid-paper-style plot of the adjusted
  traverse loop.

**Missing Line tab** — 4 modes in a 2×2 toggle grid:
1. Inverse (2 Pts) — bearing/distance between two known N/E coordinates.
2. Closing Leg — same calc as the live Traverse tab panel, shown standalone here too.
3. Radiations — bearing/distance between two points each defined by bearing+distance
   ("radiated") from the same occupied station.
4. Bearing Compare — signed angle + turn direction (CW/CCW) between two bearings.

## Known limitations / likely next steps

- **No persistence.** State lives only in JS memory; refreshing the page loses everything.
  This was intentionally deferred (Claude.ai's artifact sandbox blocks localStorage) but
  is the top priority for a real field tool — surveyors cannot lose data mid-job. Needs
  IndexedDB or localStorage, plus probably an export/import (JSON or CSV) so a job can be
  backed up or moved between devices.
- **No true PWA manifest/service worker yet** — currently works via "Add to Home Screen"
  in Chrome but isn't a real installable/offline-cached PWA. Needs `manifest.json` +
  a service worker for real offline reliability.
- The Backsight Orientation Check (Traverse tab) is currently self-contained and does NOT
  feed into the Closure tab's stats. Wiring the finishing-backsight misclosure into the
  Closure summary was discussed as a possible next step but not yet built.
- Missing Line calculators are stateless scratch pads (values aren't saved to `state`,
  just read from the DOM on "Calculate" click) — fine for a quick calculator, but worth
  knowing if you're looking for where that data "lives."
- No multi-job / multi-file support — one traverse per session.
