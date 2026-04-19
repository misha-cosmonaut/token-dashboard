# Customizations

What makes Token Dashboard different from [phuryn/claude-usage](https://github.com/phuryn/claude-usage).

This file is the running log of decisions. Update it whenever we commit to a direction so the rationale stays with the code.

## UI improvements

_TBD — user wants "better-looking" but specifics not yet defined._

Candidate directions to discuss:
- Modern design system (shadcn/ui, Radix, or custom)
- Dark/light mode
- Per-project and per-session drill-down views
- Live-updating "current session" panel
- Sparkline charts instead of only Chart.js bar/line

## Functional changes

_TBD._

Candidates:
- [ ] Configurable pricing table (handle unknown models, plan-aware cost display)
- [ ] "Subscription value" view — what you'd pay on API vs. what you pay on Pro/Max
- [ ] Session drill-down with message-level breakdown
- [ ] Per-project comparisons
- [ ] Export to CSV / JSON
- [ ] Tag / filter sessions
- [ ] Live tail of the active session

## Tech stack

_Not chosen yet._ Do not assume Python just because the inspiration uses it.

## Explicitly rejected

_(none yet — add anything we decide against, with the reason)_
