# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project Overview

**Token Dashboard** — a local dashboard for tracking Claude Code token usage, costs, and session history. Reads the JSONL transcripts Claude Code writes to `~/.claude/projects/` and turns them into charts and summaries.

Inspired by [phuryn/claude-usage](https://github.com/phuryn/claude-usage), but this project is **not** a straight clone. The goals are:

1. A better-looking, more modern UI than the original.
2. Different / expanded functionality (specifics TBD — see `docs/customizations.md`).

See `docs/inspiration.md` for a summary of what the original does and its limitations.

## Status

Empty scaffold. Tech stack, architecture, and feature set are all open decisions — do **not** assume Python + http.server just because the inspiration uses them. Ask before locking choices in.

Open questions to resolve with the user before writing code:
- Language / runtime (Python? Node + Next.js? Tauri? Electron? Pure static + local file picker?)
- UI framework and design direction
- Whether to keep SQLite or use something else
- Which limitations of the original to fix first (Cowork sessions, non-standard model cost handling, etc.)
- Whether it runs as a CLI + localhost server, a desktop app, or both

## Data Source (same as the original)

Claude Code writes one JSONL file per session to:

```
~/.claude/projects/<project-slug>/<session-id>.jsonl
```

Each line is a message record. Usage fields live at `message.usage` (input/output tokens, cache read/write counts) and `message.model` identifies the model. On this machine `~` = `C:\Users\nateh\`.

## Working Directory Note

The project path is `C:\Users\nateh\OneDrive\Desktop\Token Dashboard` — shell is bash (Git Bash / WSL-style). Use forward slashes and Unix syntax.

## Conventions (to be expanded as we build)

- Keep the dashboard fully local — no telemetry, no remote calls for user data.
- Don't over-scaffold. Add files/folders only when a feature needs them.
- Document UI/UX decisions in `docs/` as they're made so the rationale doesn't get lost.
