# Token Dashboard

A local dashboard for Claude Code token usage, costs, and session history. Reads the JSONL transcripts Claude Code writes to `~/.claude/projects/` and turns them into per-prompt cost analytics, tool/file heatmaps, subagent attribution, cache analytics, project comparisons, and a rule-based tips engine.

## Quick Start

```
git clone <url>
cd token-dashboard
python cli.py dashboard
```

Opens http://localhost:8080. **No `pip install` required** — Python 3.8+ stdlib only.

## CLI

```
python cli.py scan        # populate the local DB from JSONL
python cli.py today       # today's totals (terminal)
python cli.py stats       # all-time totals (terminal)
python cli.py tips        # active suggestions (terminal)
python cli.py dashboard   # scan + open browser
```

## What's different from `phuryn/claude-usage`

See `docs/customizations.md` for the full list. Highlights:

- Per-prompt drill-down with full text, tool calls, and result sizes
- Turn-by-turn session view
- Tool & file heatmap (which files get re-read, which bash commands repeat)
- Subagent cost attribution
- Cache hit-rate trends
- Plan-aware pricing (API / Pro / Max / Max-20x)
- Rule-based tips engine that surfaces savings
- Modern dark UI (Linear-style, JetBrains Mono numerals)

## License

MIT.
