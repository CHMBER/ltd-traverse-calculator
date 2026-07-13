# LTD Traverse Calculator

A single-file field surveying calculator (Traverse entry, Closure/Adjustment, Coordinate
plotting, Calculator tools) built for offline use on Samsung Android work phones.

## Running it

Live at **https://chmber.github.io/ltd-traverse-calculator/** (GitHub Pages, deploys
automatically from `main` — just push to update it). Open that URL on any phone and:

- **Android (Chrome)**: tap ⋮ → **Add to Home Screen**.
- **iPhone (Safari)**: tap Share → **Add to Home Screen**.

It then runs full-screen like an installed app. No build step either way — `index.html`
can also just be opened directly in a browser without any hosting at all.

## Project status

See `CLAUDE.md` for full context on what's built, conventions used, and what's next
(this file is written for Claude Code to read on startup, but useful for any contributor).

Job data now auto-saves to the browser's `localStorage` and survives a refresh — use
the header's **NEW JOB** button to clear it and start over. Top of the "next steps"
list now: JSON export/import (to back up or move a job between devices) and a real
PWA manifest/service worker for offline installs.
