# Marketing Bundle — Pre-Onboarding

Static GitHub Pages landing page that walks a Domo marketing teammate from **zero** (no Claude Code, no GitHub, no VS Code) to **ready for their onboarding meeting** with Jake.

**Live URL:** https://jakeheaps-coder.github.io/marketing-bundle-onboarding/

## What it covers (5 steps)

1. Get Claude Code access (email John Brandt for the Domo Claude Team plan)
2. Request a Domo GitHub account via ServiceNow
3. Download VS Code
4. Install the Claude Code extension + sign in
5. **Connect GitHub + install the bundle** — via the AI-driven installer (`setup.md`) which Claude Code runs end-to-end without the user opening a terminal

End state: a "Congrats — you're ready" confirmation. Teammate messages Jake; Jake schedules a 30-min follow-up that focuses on which skills to use rather than plumbing.

## `setup.md` — the AI-driven installer

A separate downloadable artifact at `/setup.md`. The teammate downloads it from Step 5, drags it into Claude Code, and types "follow this file." Claude Code then runs all 9 phases:

1. Platform sanity check
2. Homebrew install + PATH register (with `osascript` admin-GUI dialog for sudo, never Terminal)
3. Install `gh`, `git`, `jq`, `python@3.12`
4. GitHub auth via `gh auth login --web` device-code flow (browser tab only)
5. Verify repo access to `jake-heaps_domo/marketing-bundle`
6. Scaffold `~/ai_projects/` workspace
7. Clone bundle + run `install-on-laptop.sh`
8. Set up `~/ai_projects/.env` (KG key pre-filled)
9. **One** VS Code reload (Cmd+Shift+P → "Developer: Reload Window") — the only manual step
10. Post-reload sanity check + handoff to `/mkt-bundle-onboard-me`

The teammate's only direct interactions: one macOS admin password dialog, one browser tab, one VS Code reload. Never a terminal.

This is built deterministically (hardcoded paths, exact commands, STOP-and-surface fallback) because every judgment call we hand to Claude Code at install-time is a fresh-laptop trip-hazard. The five issues from the first onboarding session — stale PATH after brew, settings.json not reloaded, gh re-install loop, folder casing, no allow-list — are each preempted with hardcoded fixes.

## Design

Static HTML + embedded CSS, no framework, no build step. Design tokens and component patterns are lifted verbatim from [`jakeheaps-coder/domo-toolkit-onboarding`](https://github.com/jakeheaps-coder/domo-toolkit-onboarding) — same Domo brand colors (Domo Blue `#99CCEE`, Orange CTA `#FF9922`), Open Sans, mobile-responsive grid.

## Editing

Edit `index.html` directly. Changes pushed to `main` deploy automatically via GitHub Pages.

## Sharing

Send the URL to a teammate as soon as they're slated to join the marketing bundle so the multi-day approval flows (Steps 1 + 2) start in parallel.

## Author

Jake Heaps · jake.heaps@domo.com · Domo Marketing
