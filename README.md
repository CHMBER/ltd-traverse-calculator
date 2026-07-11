# LTD Traverse Calculator

A single-file field surveying calculator (Traverse entry, Closure/Adjustment, Coordinate
plotting, Calculator tools) built for offline use on Samsung Android work phones.

## Running it

No build step. Just open `index.html` in a browser, or on a phone:

1. Host the folder somewhere reachable (or just open the file directly in Chrome).
2. In Chrome, tap ⋮ → **Add to Home Screen**.
3. It runs full-screen like an installed app, no internet required.

## Project status

See `CLAUDE.md` for full context on what's built, conventions used, and what's next
(this file is written for Claude Code to read on startup, but useful for any contributor).

Job data now auto-saves to the browser's `localStorage` and survives a refresh — use
the header's **NEW JOB** button to clear it and start over. Top of the "next steps"
list now: JSON export/import (to back up or move a job between devices) and a real
PWA manifest/service worker for offline installs.
