# Marketing Bundle — Pre-Onboarding

Static GitHub Pages landing page that walks a Domo marketing teammate from **zero** (no Claude Code, no GitHub, no VS Code) to **ready for their onboarding meeting** with Jake.

**Live URL:** https://jakeheaps-coder.github.io/marketing-bundle-onboarding/

## What it covers (5 steps)

1. Get Claude Code access (email John Brandt for the Domo Claude Team plan)
2. Request a Domo GitHub account via ServiceNow
3. Download VS Code
4. Install the Claude Code extension + sign in
5. Connect GitHub via the `gh` CLI

End state: a "Congrats — you're ready" confirmation. Teammate messages Jake; Jake schedules the 60-min onboarding session where they install the bundle and ship their first PR.

## Design

Static HTML + embedded CSS, no framework, no build step. Design tokens and component patterns are lifted verbatim from [`jakeheaps-coder/domo-toolkit-onboarding`](https://github.com/jakeheaps-coder/domo-toolkit-onboarding) — same Domo brand colors (Domo Blue `#99CCEE`, Orange CTA `#FF9922`), Open Sans, mobile-responsive grid.

## Editing

Edit `index.html` directly. Changes pushed to `main` deploy automatically via GitHub Pages.

## Sharing

Send the URL to a teammate as soon as they're slated to join the marketing bundle so the multi-day approval flows (Steps 1 + 2) start in parallel.

## Author

Jake Heaps · jake.heaps@domo.com · Domo Marketing
