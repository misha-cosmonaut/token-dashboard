# Token Dashboard v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local Claude Code usage dashboard (Python stdlib backend + single-page dark UI) that surfaces per-prompt cost, tool/file heatmaps, subagent attribution, cache analytics, project comparison, and a rule-based tips engine.

**Architecture:** Single-process Python `http.server` reads JSONL transcripts under `~/.claude/projects/`, normalizes them into a SQLite DB at `~/.claude/token-dashboard.db`, and serves a vanilla-JS SPA from `web/`. SSE pushes diffs after each rescan. No `pip install`, no `npm install`.

**Tech Stack:** Python 3.8+ stdlib (`sqlite3`, `http.server`, `json`, `pathlib`, `argparse`); vanilla JS modules; hand-tuned CSS; ECharts via CDN; Inter + JetBrains Mono via Google Fonts.

**Source spec:** `C:\Users\nateh\.claude\plans\so-right-now-what-nifty-wolf.md`

---

## File map

| Path | Responsibility |
|---|---|
| `cli.py` | argparse entrypoint; dispatches `scan`, `today`, `stats`, `tips`, `dashboard` |
| `pricing.json` | Model rate table (per-1M tokens) and plan monthly costs |
| `token_dashboard/__init__.py` | Package marker (empty) |
| `token_dashboard/db.py` | SQLite schema, connection helper, query functions |
| `token_dashboard/scanner.py` | JSONL walker + per-record parser; writes to DB |
| `token_dashboard/pricing.py` | `cost_for(model, usage)`, `format_for_user(cost)`, plan I/O |
| `token_dashboard/tips.py` | Four rule functions + dismissal tracking |
| `token_dashboard/server.py` | HTTP server, JSON endpoints, SSE diff stream, static |
| `web/index.html` | Single-page shell with tab nav and root `<main id="app">` |
| `web/style.css` | Linear-style dark theme |
| `web/app.js` | Hash router, fetch helpers, shared state |
| `web/charts.js` | ECharts wrappers (line, bar, donut) |
| `web/routes/{overview,prompts,sessions,tools,projects,tips,settings}.js` | One per tab |
| `tests/fixtures/*.jsonl` | Tiny sample sessions for scanner tests |
| `tests/test_*.py` | Pytest suites — but use `unittest` to avoid third-party dep |

---

## Task 1: Scaffold + git init + first commit

**Files:**
- Create: `C:/Users/nateh/OneDrive/Desktop/Token Dashboard/.gitignore`
- Create: `C:/Users/nateh/OneDrive/Desktop/Token Dashboard/README.md`
- Create: `C:/Users/nateh/OneDrive/Desktop/Token Dashboard/pricing.json`
- Create: `C:/Users/nateh/OneDrive/Desktop/Token Dashboard/token_dashboard/__init__.py`

- [ ] **Step 1: Initialize git and create .gitignore**

```bash
cd "/c/Users/nateh/OneDrive/Desktop/Token Dashboard" && git init
```

Write `.gitignore`:
```
__pycache__/
*.pyc
.venv/
.DS_Store
out/
node_modules/
*.db
.superpowers/
```

- [ ] **Step 2: Write README.md**

```markdown
# Token Dashboard

A local dashboard for Claude Code token usage, costs, and session history.

## Quick Start

```
git clone <url>
cd token-dashboard
python cli.py dashboard
```

Opens http://localhost:8080. No `pip install` required — Python 3.8+ stdlib only.

See `docs/customizations.md` for what differs from `phuryn/claude-usage`.
```

- [ ] **Step 3: Write pricing.json**

```json
{
  "models": {
    "claude-opus-4-7":     { "tier": "opus",   "input": 15.00, "output": 75.00, "cache_read": 1.50, "cache_create_5m": 18.75, "cache_create_1h": 30.00 },
    "claude-opus-4-6":     { "tier": "opus",   "input": 15.00, "output": 75.00, "cache_read": 1.50, "cache_create_5m": 18.75, "cache_create_1h": 30.00 },
    "claude-sonnet-4-6":   { "tier": "sonnet", "input":  3.00, "output": 15.00, "cache_read": 0.30, "cache_create_5m":  3.75, "cache_create_1h":  6.00 },
    "claude-haiku-4-5":    { "tier": "haiku",  "input":  1.00, "output":  5.00, "cache_read": 0.10, "cache_create_5m":  1.25, "cache_create_1h":  2.00 }
  },
  "tier_fallback": {
    "opus":   { "input": 15.00, "output": 75.00, "cache_read": 1.50, "cache_create_5m": 18.75, "cache_create_1h": 30.00 },
    "sonnet": { "input":  3.00, "output": 15.00, "cache_read": 0.30, "cache_create_5m":  3.75, "cache_create_1h":  6.00 },
    "haiku":  { "input":  1.00, "output":  5.00, "cache_read": 0.10, "cache_create_5m":  1.25, "cache_create_1h":  2.00 }
  },
  "plans": {
    "api":     { "monthly": 0,   "label": "API (pay-per-token)" },
    "pro":     { "monthly": 20,  "label": "Pro" },
    "max":     { "monthly": 100, "label": "Max" },
    "max-20x": { "monthly": 200, "label": "Max 20x" }
  }
}
```

> **Note:** Rates are per **1M tokens**, USD. If Anthropic publishes new tiers, edit this file.

- [ ] **Step 4: Create empty package**

Write `token_dashboard/__init__.py`:
```python
"""Token Dashboard — local Claude Code usage analytics."""
__version__ = "0.1.0"
```

- [ ] **Step 5: Commit**

```bash
git add .gitignore README.md pricing.json token_dashboard/__init__.py
git commit -m "chore: scaffold project (gitignore, readme, pricing, package)"
```

---

## Task 2: db.py — schema + init

**Files:**
- Create: `token_dashboard/db.py`
- Create: `tests/test_db.py`

- [ ] **Step 1: Write the failing test**

`tests/test_db.py`:
```python
import os
import sqlite3
import tempfile
import unittest
from token_dashboard.db import init_db, connect

class InitDbTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.mkdtemp()
        self.db_path = os.path.join(self.tmp, "test.db")

    def test_init_creates_expected_tables(self):
        init_db(self.db_path)
        with sqlite3.connect(self.db_path) as c:
            tables = {r[0] for r in c.execute("SELECT name FROM sqlite_master WHERE type='table'")}
        self.assertEqual(
            tables,
            {"files", "messages", "tool_calls", "plan", "dismissed_tips"},
        )

    def test_init_is_idempotent(self):
        init_db(self.db_path)
        init_db(self.db_path)  # must not raise

    def test_connect_returns_row_factory(self):
        init_db(self.db_path)
        with connect(self.db_path) as c:
            r = c.execute("SELECT 1 AS one").fetchone()
        self.assertEqual(r["one"], 1)

if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m unittest tests.test_db -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'token_dashboard.db'`

- [ ] **Step 3: Implement db.py**

```python
"""SQLite schema, connection, and shared query helpers."""
from __future__ import annotations
import sqlite3
from contextlib import contextmanager
from pathlib import Path

SCHEMA = """
CREATE TABLE IF NOT EXISTS files (
  path        TEXT PRIMARY KEY,
  mtime       REAL    NOT NULL,
  bytes_read  INTEGER NOT NULL,
  scanned_at  REAL    NOT NULL
);

CREATE TABLE IF NOT EXISTS messages (
  uuid                    TEXT PRIMARY KEY,
  parent_uuid             TEXT,
  session_id              TEXT NOT NULL,
  project_slug            TEXT NOT NULL,
  cwd                     TEXT,
  git_branch              TEXT,
  cc_version              TEXT,
  entrypoint              TEXT,
  type                    TEXT NOT NULL,
  is_sidechain            INTEGER NOT NULL DEFAULT 0,
  agent_id                TEXT,
  timestamp               TEXT NOT NULL,
  model                   TEXT,
  stop_reason             TEXT,
  prompt_id               TEXT,
  input_tokens            INTEGER NOT NULL DEFAULT 0,
  output_tokens           INTEGER NOT NULL DEFAULT 0,
  cache_read_tokens       INTEGER NOT NULL DEFAULT 0,
  cache_create_5m_tokens  INTEGER NOT NULL DEFAULT 0,
  cache_create_1h_tokens  INTEGER NOT NULL DEFAULT 0,
  prompt_text             TEXT,
  prompt_chars            INTEGER,
  tool_calls_json         TEXT
);
CREATE INDEX IF NOT EXISTS idx_messages_session   ON messages(session_id);
CREATE INDEX IF NOT EXISTS idx_messages_project   ON messages(project_slug);
CREATE INDEX IF NOT EXISTS idx_messages_timestamp ON messages(timestamp);
CREATE INDEX IF NOT EXISTS idx_messages_model     ON messages(model);

CREATE TABLE IF NOT EXISTS tool_calls (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  message_uuid  TEXT    NOT NULL,
  session_id    TEXT    NOT NULL,
  project_slug  TEXT    NOT NULL,
  tool_name     TEXT    NOT NULL,
  target        TEXT,
  result_tokens INTEGER,
  is_error      INTEGER NOT NULL DEFAULT 0,
  timestamp     TEXT    NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_tools_session ON tool_calls(session_id);
CREATE INDEX IF NOT EXISTS idx_tools_name    ON tool_calls(tool_name);
CREATE INDEX IF NOT EXISTS idx_tools_target  ON tool_calls(target);

CREATE TABLE IF NOT EXISTS plan (
  k TEXT PRIMARY KEY,
  v TEXT
);

CREATE TABLE IF NOT EXISTS dismissed_tips (
  tip_key       TEXT PRIMARY KEY,
  dismissed_at  REAL NOT NULL
);
"""

def default_db_path() -> Path:
    return Path.home() / ".claude" / "token-dashboard.db"

def init_db(path: str | Path) -> None:
    path = Path(path)
    path.parent.mkdir(parents=True, exist_ok=True)
    with sqlite3.connect(path) as c:
        c.executescript(SCHEMA)

@contextmanager
def connect(path: str | Path):
    conn = sqlite3.connect(path)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    try:
        yield conn
    finally:
        conn.close()
```

- [ ] **Step 4: Run tests, expect PASS**

Run: `python -m unittest tests.test_db -v`
Expected: 3 tests pass.

- [ ] **Step 5: Commit**

```bash
git add token_dashboard/db.py tests/test_db.py
git commit -m "feat(db): schema + connect helper with idempotent init"
```

---

## Task 3: scanner.py — parse a single record

**Files:**
- Create: `token_dashboard/scanner.py`
- Create: `tests/fixtures/simple_assistant.json`
- Create: `tests/test_scanner_parse.py`

- [ ] **Step 1: Write fixture (single assistant record)**

`tests/fixtures/simple_assistant.json` (one JSON object — fixture is not JSONL because we test the parser, not the file walker):
```json
{
  "type": "assistant",
  "uuid": "msg-1",
  "parentUuid": "user-1",
  "sessionId": "sess-1",
  "cwd": "C:/work",
  "gitBranch": "main",
  "version": "2.1.98",
  "entrypoint": "cli",
  "isSidechain": false,
  "timestamp": "2026-04-10T12:00:00.000Z",
  "requestId": "req-1",
  "userType": "external",
  "message": {
    "id": "msg_x",
    "role": "assistant",
    "model": "claude-opus-4-7",
    "stop_reason": "end_turn",
    "content": [{ "type": "text", "text": "ok" }],
    "usage": {
      "input_tokens": 10,
      "output_tokens": 5,
      "cache_read_input_tokens": 100,
      "cache_creation_input_tokens": 50,
      "cache_creation": { "ephemeral_5m_input_tokens": 30, "ephemeral_1h_input_tokens": 20 }
    }
  }
}
```

- [ ] **Step 2: Write the failing test**

`tests/test_scanner_parse.py`:
```python
import json
import os
import unittest
from token_dashboard.scanner import parse_record

FIXTURE = os.path.join(os.path.dirname(__file__), "fixtures", "simple_assistant.json")

class ParseRecordTests(unittest.TestCase):
    def setUp(self):
        with open(FIXTURE) as f:
            self.raw = json.load(f)

    def test_parses_assistant_usage(self):
        msg, tools = parse_record(self.raw, project_slug="proj-x")
        self.assertEqual(msg["uuid"], "msg-1")
        self.assertEqual(msg["session_id"], "sess-1")
        self.assertEqual(msg["project_slug"], "proj-x")
        self.assertEqual(msg["model"], "claude-opus-4-7")
        self.assertEqual(msg["input_tokens"], 10)
        self.assertEqual(msg["output_tokens"], 5)
        self.assertEqual(msg["cache_read_tokens"], 100)
        self.assertEqual(msg["cache_create_5m_tokens"], 30)
        self.assertEqual(msg["cache_create_1h_tokens"], 20)
        self.assertEqual(msg["is_sidechain"], 0)
        self.assertIsNone(msg["agent_id"])
        self.assertEqual(tools, [])

if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 3: Run, verify FAIL**

Run: `python -m unittest tests.test_scanner_parse -v`
Expected: `ModuleNotFoundError`.

- [ ] **Step 4: Implement parse_record**

`token_dashboard/scanner.py`:
```python
"""JSONL transcript walker + parser."""
from __future__ import annotations
import json
from typing import Any, Iterable

def _usage(rec: dict) -> dict:
    u = (rec.get("message") or {}).get("usage") or {}
    cc = u.get("cache_creation") or {}
    return {
        "input_tokens":           int(u.get("input_tokens") or 0),
        "output_tokens":          int(u.get("output_tokens") or 0),
        "cache_read_tokens":      int(u.get("cache_read_input_tokens") or 0),
        "cache_create_5m_tokens": int(cc.get("ephemeral_5m_input_tokens") or 0),
        "cache_create_1h_tokens": int(cc.get("ephemeral_1h_input_tokens") or 0),
    }

def _prompt_text(rec: dict) -> tuple[str | None, int | None]:
    if rec.get("type") != "user":
        return None, None
    content = (rec.get("message") or {}).get("content")
    if isinstance(content, str):
        return content, len(content)
    if isinstance(content, list):
        parts = [b.get("text", "") for b in content if isinstance(b, dict) and b.get("type") == "text"]
        text = "".join(parts) if parts else None
        return text, len(text) if text else None
    return None, None

def parse_record(rec: dict, project_slug: str) -> tuple[dict, list[dict]]:
    """Return (message_row, [tool_call_rows]). Tool extraction lives in next task."""
    msg_obj = rec.get("message") or {}
    text, chars = _prompt_text(rec)
    msg = {
        "uuid":         rec.get("uuid"),
        "parent_uuid":  rec.get("parentUuid"),
        "session_id":   rec.get("sessionId"),
        "project_slug": project_slug,
        "cwd":          rec.get("cwd"),
        "git_branch":   rec.get("gitBranch"),
        "cc_version":   rec.get("version"),
        "entrypoint":   rec.get("entrypoint"),
        "type":         rec.get("type"),
        "is_sidechain": 1 if rec.get("isSidechain") else 0,
        "agent_id":     rec.get("agentId"),
        "timestamp":    rec.get("timestamp"),
        "model":        msg_obj.get("model"),
        "stop_reason":  msg_obj.get("stop_reason"),
        "prompt_id":    rec.get("promptId"),
        "prompt_text":  text,
        "prompt_chars": chars,
        "tool_calls_json": None,
        **_usage(rec),
    }
    return msg, []
```

- [ ] **Step 5: Run tests, expect PASS**

Run: `python -m unittest tests.test_scanner_parse -v`

- [ ] **Step 6: Commit**

```bash
git add token_dashboard/scanner.py tests/fixtures/simple_assistant.json tests/test_scanner_parse.py
git commit -m "feat(scanner): parse a single record with usage extraction"
```

---

## Task 4: scanner.py — extract tool calls

**Files:**
- Create: `tests/fixtures/tool_use_assistant.json`
- Create: `tests/fixtures/tool_result_user.json`
- Modify: `token_dashboard/scanner.py`
- Modify: `tests/test_scanner_parse.py`

- [ ] **Step 1: Add fixtures**

`tests/fixtures/tool_use_assistant.json`:
```json
{
  "type": "assistant",
  "uuid": "msg-2",
  "sessionId": "sess-1",
  "timestamp": "2026-04-10T12:00:01.000Z",
  "isSidechain": false,
  "message": {
    "model": "claude-sonnet-4-6",
    "content": [
      { "type": "text", "text": "checking" },
      { "type": "tool_use", "id": "tu1", "name": "Read", "input": { "file_path": "C:/proj/foo.py" } },
      { "type": "tool_use", "id": "tu2", "name": "Bash", "input": { "command": "npm run lint" } }
    ],
    "usage": { "input_tokens": 1, "output_tokens": 1 }
  }
}
```

`tests/fixtures/tool_result_user.json`:
```json
{
  "type": "user",
  "uuid": "msg-3",
  "sessionId": "sess-1",
  "timestamp": "2026-04-10T12:00:02.000Z",
  "isSidechain": false,
  "message": {
    "role": "user",
    "content": [
      { "type": "tool_result", "tool_use_id": "tu1", "content": "x" * 0, "is_error": false }
    ]
  }
}
```

(JSON doesn't allow `* 0`; replace with empty string `""`.)

- [ ] **Step 2: Add tests**

Append to `tests/test_scanner_parse.py`:
```python
class ToolExtractionTests(unittest.TestCase):
    def _load(self, name):
        with open(os.path.join(os.path.dirname(__file__), "fixtures", name)) as f:
            return json.load(f)

    def test_extracts_tool_uses(self):
        rec = self._load("tool_use_assistant.json")
        msg, tools = parse_record(rec, project_slug="p")
        self.assertEqual(len(tools), 2)
        names = [t["tool_name"] for t in tools]
        self.assertEqual(names, ["Read", "Bash"])
        self.assertEqual(tools[0]["target"], "C:/proj/foo.py")
        self.assertEqual(tools[1]["target"], "npm run lint")
        self.assertEqual(msg["tool_calls_json"], '[{"name": "Read", "target": "C:/proj/foo.py"}, {"name": "Bash", "target": "npm run lint"}]')
```

- [ ] **Step 3: Implement target extraction**

Modify `scanner.py`:
```python
_TARGET_FIELDS = {
    "Read":      "file_path",
    "Edit":      "file_path",
    "Write":     "file_path",
    "Glob":      "pattern",
    "Grep":      "pattern",
    "Bash":      "command",
    "WebFetch":  "url",
    "WebSearch": "query",
    "Task":      "subagent_type",
    "Skill":     "skill",
}

def _target(name: str, inp: dict) -> str | None:
    field = _TARGET_FIELDS.get(name)
    if field and isinstance(inp, dict):
        v = inp.get(field)
        if isinstance(v, str):
            return v[:500]
    return None

def _extract_tools(rec: dict) -> list[dict]:
    out = []
    content = (rec.get("message") or {}).get("content")
    if not isinstance(content, list):
        return out
    for block in content:
        if not isinstance(block, dict) or block.get("type") != "tool_use":
            continue
        name = block.get("name") or "unknown"
        target = _target(name, block.get("input") or {})
        out.append({
            "tool_name": name,
            "target":    target,
            "is_error":  0,
            "timestamp": rec.get("timestamp"),
        })
    return out
```

In `parse_record`, replace the tool_calls fields:
```python
    tools = _extract_tools(rec)
    msg["tool_calls_json"] = json.dumps([{"name": t["tool_name"], "target": t["target"]} for t in tools]) if tools else None
    for t in tools:
        t["message_uuid"] = msg["uuid"]
        t["session_id"]   = msg["session_id"]
        t["project_slug"] = project_slug
        t["result_tokens"] = None
    return msg, tools
```

- [ ] **Step 4: Run tests, expect PASS**

Run: `python -m unittest tests.test_scanner_parse -v`

- [ ] **Step 5: Commit**

```bash
git add token_dashboard/scanner.py tests/fixtures/tool_use_assistant.json tests/fixtures/tool_result_user.json tests/test_scanner_parse.py
git commit -m "feat(scanner): extract tool_use blocks with per-tool target field"
```

---

## Task 5: scanner.py — sidechain + result token estimation

**Files:**
- Modify: `token_dashboard/scanner.py`
- Modify: `tests/test_scanner_parse.py`

- [ ] **Step 1: Add sidechain test**

```python
class SidechainTests(unittest.TestCase):
    def test_is_sidechain_flag_propagates(self):
        rec = {
            "type": "assistant", "uuid": "u", "sessionId": "s",
            "timestamp": "t", "isSidechain": True, "agentId": "agent-explore-1",
            "message": {"model": "claude-sonnet-4-6", "usage": {"input_tokens": 1, "output_tokens": 1}}
        }
        msg, _ = parse_record(rec, project_slug="p")
        self.assertEqual(msg["is_sidechain"], 1)
        self.assertEqual(msg["agent_id"], "agent-explore-1")

    def test_tool_result_estimates_tokens(self):
        rec = {
            "type": "user", "uuid": "u2", "sessionId": "s",
            "timestamp": "t", "isSidechain": False,
            "message": {"role": "user", "content": [
                {"type": "tool_result", "tool_use_id": "tu1", "content": "x" * 4000, "is_error": False}
            ]}
        }
        msg, tools = parse_record(rec, project_slug="p")
        self.assertEqual(msg["type"], "user")
        # tool_result rows are returned for tracking even though they aren't tool_uses
        self.assertEqual(len(tools), 1)
        self.assertEqual(tools[0]["tool_name"], "_tool_result")
        self.assertAlmostEqual(tools[0]["result_tokens"], 1000, delta=10)
```

- [ ] **Step 2: Implement tool_result handling**

Add to `scanner.py`:
```python
def _extract_results(rec: dict) -> list[dict]:
    out = []
    content = (rec.get("message") or {}).get("content")
    if not isinstance(content, list):
        return out
    for block in content:
        if not isinstance(block, dict) or block.get("type") != "tool_result":
            continue
        body = block.get("content")
        chars = len(body) if isinstance(body, str) else (sum(len(p.get("text","")) for p in body if isinstance(p, dict)) if isinstance(body, list) else 0)
        out.append({
            "tool_name":    "_tool_result",
            "target":       block.get("tool_use_id"),
            "is_error":     1 if block.get("is_error") else 0,
            "result_tokens": chars // 4,
            "timestamp":    rec.get("timestamp"),
        })
    return out
```

In `parse_record`, after `tools = _extract_tools(rec)`:
```python
    tools.extend(_extract_results(rec))
```

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_scanner_parse -v`

- [ ] **Step 4: Commit**

```bash
git add token_dashboard/scanner.py tests/test_scanner_parse.py
git commit -m "feat(scanner): handle sidechain attribution and tool_result token estimation"
```

---

## Task 6: scanner.py — file walk + DB upsert

**Files:**
- Modify: `token_dashboard/scanner.py`
- Create: `tests/fixtures/sample_session.jsonl` (3 lines: user, assistant, tool_result)
- Create: `tests/test_scanner_walk.py`

- [ ] **Step 1: Build the fixture session**

`tests/fixtures/sample_session.jsonl` (one record per line):
```
{"type":"user","uuid":"u1","sessionId":"s1","timestamp":"2026-04-10T00:00:00Z","isSidechain":false,"cwd":"/c/work","gitBranch":"main","version":"2.1.98","entrypoint":"cli","message":{"role":"user","content":"plan the refactor"}}
{"type":"assistant","uuid":"a1","parentUuid":"u1","sessionId":"s1","timestamp":"2026-04-10T00:00:01Z","isSidechain":false,"message":{"model":"claude-opus-4-7","content":[{"type":"text","text":"ok"},{"type":"tool_use","id":"tu1","name":"Read","input":{"file_path":"foo.py"}}],"usage":{"input_tokens":10,"output_tokens":5,"cache_read_input_tokens":50}}}
{"type":"user","uuid":"u2","sessionId":"s1","timestamp":"2026-04-10T00:00:02Z","isSidechain":false,"message":{"role":"user","content":[{"type":"tool_result","tool_use_id":"tu1","content":"file contents","is_error":false}]}}
```

- [ ] **Step 2: Add walker test**

`tests/test_scanner_walk.py`:
```python
import os, sqlite3, tempfile, unittest, shutil
from token_dashboard.db import init_db
from token_dashboard.scanner import scan_dir

FIXTURE_DIR = os.path.join(os.path.dirname(__file__), "fixtures")

class WalkTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.mkdtemp()
        self.db = os.path.join(self.tmp, "t.db")
        # Mirror the projects dir layout: fixtures/<project-slug>/<file>.jsonl
        self.proj_root = os.path.join(self.tmp, "projects")
        proj_dir = os.path.join(self.proj_root, "C--work-sample")
        os.makedirs(proj_dir)
        shutil.copy(os.path.join(FIXTURE_DIR, "sample_session.jsonl"),
                    os.path.join(proj_dir, "s1.jsonl"))
        init_db(self.db)

    def test_scan_writes_messages_and_tools(self):
        n = scan_dir(self.proj_root, self.db)
        self.assertEqual(n["messages"], 3)
        self.assertEqual(n["tools"], 2)  # 1 tool_use + 1 tool_result
        with sqlite3.connect(self.db) as c:
            row = c.execute("SELECT project_slug FROM messages WHERE uuid='u1'").fetchone()
        self.assertEqual(row[0], "C--work-sample")
```

- [ ] **Step 3: Implement scan_dir**

Add to `scanner.py`:
```python
import os
from pathlib import Path
from .db import connect

INSERT_MSG = """
INSERT OR REPLACE INTO messages (
  uuid, parent_uuid, session_id, project_slug, cwd, git_branch, cc_version, entrypoint,
  type, is_sidechain, agent_id, timestamp, model, stop_reason, prompt_id,
  input_tokens, output_tokens, cache_read_tokens, cache_create_5m_tokens, cache_create_1h_tokens,
  prompt_text, prompt_chars, tool_calls_json
) VALUES (
  :uuid, :parent_uuid, :session_id, :project_slug, :cwd, :git_branch, :cc_version, :entrypoint,
  :type, :is_sidechain, :agent_id, :timestamp, :model, :stop_reason, :prompt_id,
  :input_tokens, :output_tokens, :cache_read_tokens, :cache_create_5m_tokens, :cache_create_1h_tokens,
  :prompt_text, :prompt_chars, :tool_calls_json
)
"""
INSERT_TOOL = """
INSERT INTO tool_calls (message_uuid, session_id, project_slug, tool_name, target, result_tokens, is_error, timestamp)
VALUES (:message_uuid, :session_id, :project_slug, :tool_name, :target, :result_tokens, :is_error, :timestamp)
"""

def _project_slug(file_path: Path, projects_root: Path) -> str:
    rel = file_path.relative_to(projects_root)
    # First path part is the project dir
    return rel.parts[0]

def scan_file(path: Path, project_slug: str, conn) -> dict:
    msgs = tools = 0
    with open(path, "r", encoding="utf-8", errors="replace") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                rec = json.loads(line)
            except json.JSONDecodeError:
                continue
            if not isinstance(rec, dict) or "uuid" not in rec or "type" not in rec:
                continue
            msg, tlist = parse_record(rec, project_slug)
            conn.execute(INSERT_MSG, msg)
            for t in tlist:
                conn.execute(INSERT_TOOL, t)
                tools += 1
            msgs += 1
    return {"messages": msgs, "tools": tools}

def scan_dir(projects_root: str | Path, db_path: str | Path) -> dict:
    root = Path(projects_root)
    totals = {"messages": 0, "tools": 0, "files": 0}
    with connect(db_path) as conn:
        for p in root.rglob("*.jsonl"):
            slug = _project_slug(p, root)
            sub = scan_file(p, slug, conn)
            totals["messages"] += sub["messages"]
            totals["tools"]    += sub["tools"]
            totals["files"]    += 1
        conn.commit()
    return totals
```

- [ ] **Step 4: Run tests, expect PASS**

Run: `python -m unittest tests.test_scanner_walk -v`

- [ ] **Step 5: Commit**

```bash
git add token_dashboard/scanner.py tests/fixtures/sample_session.jsonl tests/test_scanner_walk.py
git commit -m "feat(scanner): walk projects dir and upsert messages + tool calls"
```

---

## Task 7: scanner.py — incremental scan via mtime/size

**Files:**
- Modify: `token_dashboard/scanner.py`
- Modify: `tests/test_scanner_walk.py`

- [ ] **Step 1: Add incremental test**

```python
    def test_rescan_skips_unchanged_files(self):
        n1 = scan_dir(self.proj_root, self.db)
        n2 = scan_dir(self.proj_root, self.db)
        self.assertEqual(n1["messages"], 3)
        self.assertEqual(n2["messages"], 0)  # nothing new

    def test_rescan_picks_up_appended_lines(self):
        scan_dir(self.proj_root, self.db)
        path = os.path.join(self.proj_root, "C--work-sample", "s1.jsonl")
        with open(path, "a", encoding="utf-8") as f:
            f.write('\n{"type":"assistant","uuid":"a2","sessionId":"s1","timestamp":"2026-04-10T00:00:03Z","isSidechain":false,"message":{"model":"claude-haiku-4-5","usage":{"input_tokens":1,"output_tokens":1}}}')
        n2 = scan_dir(self.proj_root, self.db)
        self.assertEqual(n2["messages"], 1)
```

- [ ] **Step 2: Add files-table tracking**

In `scan_dir`, replace the loop:
```python
        for p in root.rglob("*.jsonl"):
            try:
                stat = p.stat()
            except OSError:
                continue
            row = conn.execute("SELECT mtime, bytes_read FROM files WHERE path=?", (str(p),)).fetchone()
            offset = 0
            if row and row["mtime"] == stat.st_mtime and row["bytes_read"] == stat.st_size:
                continue  # unchanged
            if row and stat.st_size > row["bytes_read"]:
                offset = row["bytes_read"]  # append-only fast path
            slug = _project_slug(p, root)
            sub = scan_file(p, slug, conn, start_byte=offset)
            conn.execute(
                "INSERT OR REPLACE INTO files (path, mtime, bytes_read, scanned_at) VALUES (?, ?, ?, ?)",
                (str(p), stat.st_mtime, stat.st_size, time.time()),
            )
            totals["messages"] += sub["messages"]
            totals["tools"]    += sub["tools"]
            totals["files"]    += 1
```

Update `scan_file` signature and seek:
```python
def scan_file(path: Path, project_slug: str, conn, start_byte: int = 0) -> dict:
    msgs = tools = 0
    with open(path, "rb") as fb:
        fb.seek(start_byte)
        for raw in fb:
            try:
                line = raw.decode("utf-8", errors="replace").strip()
            except Exception:
                continue
            if not line:
                continue
            ...
```

Add `import time` at the top.

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_scanner_walk -v`

- [ ] **Step 4: Commit**

```bash
git add token_dashboard/scanner.py tests/test_scanner_walk.py
git commit -m "feat(scanner): incremental scan with mtime + append-only resume"
```

---

## Task 8: pricing.py — cost calculation

**Files:**
- Create: `token_dashboard/pricing.py`
- Create: `tests/test_pricing.py`

- [ ] **Step 1: Failing test**

```python
import os, json, unittest, tempfile, shutil
from token_dashboard.pricing import load_pricing, cost_for, format_for_user, get_plan, set_plan
from token_dashboard.db import init_db

PRICING = os.path.abspath(os.path.join(os.path.dirname(__file__), "..", "pricing.json"))

class CostTests(unittest.TestCase):
    def setUp(self):
        self.p = load_pricing(PRICING)

    def test_known_opus_cost(self):
        u = {"input_tokens": 1_000_000, "output_tokens": 0, "cache_read_tokens": 0,
             "cache_create_5m_tokens": 0, "cache_create_1h_tokens": 0}
        c = cost_for("claude-opus-4-7", u, self.p)
        self.assertAlmostEqual(c["usd"], 15.00, places=4)
        self.assertFalse(c["estimated"])

    def test_unknown_opus_falls_back(self):
        u = {"input_tokens": 1_000_000, "output_tokens": 0, "cache_read_tokens": 0,
             "cache_create_5m_tokens": 0, "cache_create_1h_tokens": 0}
        c = cost_for("claude-opus-9-9-experimental", u, self.p)
        self.assertAlmostEqual(c["usd"], 15.00, places=4)
        self.assertTrue(c["estimated"])

    def test_unknown_unparseable(self):
        c = cost_for("custom-local-model", {"input_tokens": 9999, "output_tokens": 0,
            "cache_read_tokens": 0, "cache_create_5m_tokens": 0, "cache_create_1h_tokens": 0}, self.p)
        self.assertIsNone(c["usd"])
```

- [ ] **Step 2: Implement pricing.py**

```python
"""Pricing table + plan-aware cost formatting."""
from __future__ import annotations
import json
from pathlib import Path
from .db import connect

def load_pricing(path: str | Path) -> dict:
    return json.loads(Path(path).read_text())

def _tier_from_name(model: str) -> str | None:
    m = (model or "").lower()
    for tier in ("opus", "sonnet", "haiku"):
        if tier in m:
            return tier
    return None

def cost_for(model: str, usage: dict, pricing: dict) -> dict:
    """Return {usd, estimated, breakdown}. usd=None when no tier match."""
    rates = pricing["models"].get(model)
    estimated = False
    if rates is None:
        tier = _tier_from_name(model)
        if tier and tier in pricing["tier_fallback"]:
            rates = pricing["tier_fallback"][tier]
            estimated = True
        else:
            return {"usd": None, "estimated": True, "breakdown": {}}
    bd = {
        "input":           usage["input_tokens"]            * rates["input"]           / 1_000_000,
        "output":          usage["output_tokens"]           * rates["output"]          / 1_000_000,
        "cache_read":      usage["cache_read_tokens"]       * rates["cache_read"]      / 1_000_000,
        "cache_create_5m": usage["cache_create_5m_tokens"]  * rates["cache_create_5m"] / 1_000_000,
        "cache_create_1h": usage["cache_create_1h_tokens"]  * rates["cache_create_1h"] / 1_000_000,
    }
    return {"usd": round(sum(bd.values()), 6), "estimated": estimated, "breakdown": bd}

def get_plan(db_path: str | Path, default: str = "api") -> str:
    with connect(db_path) as c:
        row = c.execute("SELECT v FROM plan WHERE k='plan'").fetchone()
    return row["v"] if row else default

def set_plan(db_path: str | Path, plan: str) -> None:
    with connect(db_path) as c:
        c.execute("INSERT OR REPLACE INTO plan (k, v) VALUES ('plan', ?)", (plan,))
        c.commit()

def format_for_user(api_cost_usd: float, plan: str, pricing: dict) -> dict:
    p = pricing["plans"].get(plan, pricing["plans"]["api"])
    if plan == "api" or p["monthly"] == 0:
        return {"display_usd": api_cost_usd, "subtitle": None, "subscription_usd": None}
    return {
        "display_usd":      api_cost_usd,
        "subtitle":         f"You pay ${p['monthly']}/mo on {p['label']}",
        "subscription_usd": p["monthly"],
    }
```

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_pricing -v`

- [ ] **Step 4: Commit**

```bash
git add token_dashboard/pricing.py tests/test_pricing.py
git commit -m "feat(pricing): cost_for with tier fallback + plan-aware formatting"
```

---

## Task 9: db.py — query helpers

**Files:**
- Modify: `token_dashboard/db.py`
- Create: `tests/test_queries.py`

- [ ] **Step 1: Failing test**

```python
import os, sqlite3, tempfile, unittest
from token_dashboard.db import init_db, connect
from token_dashboard.db import (
    overview_totals, expensive_prompts, project_summary,
    tool_token_breakdown, recent_sessions, session_turns
)

class QueryTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.mkdtemp()
        self.db = os.path.join(self.tmp, "q.db")
        init_db(self.db)
        with connect(self.db) as c:
            c.executescript("""
            INSERT INTO messages (uuid, session_id, project_slug, type, timestamp, model,
              input_tokens, output_tokens, cache_read_tokens, cache_create_5m_tokens, cache_create_1h_tokens,
              prompt_text, prompt_chars)
            VALUES
              ('u1','s1','projA','user','2026-04-10T00:00:00Z',NULL,0,0,0,0,0,'big prompt',10),
              ('a1','s1','projA','assistant','2026-04-10T00:00:01Z','claude-opus-4-7',100,200,300,0,0,NULL,NULL),
              ('u2','s2','projB','user','2026-04-11T00:00:00Z',NULL,0,0,0,0,0,'small',5),
              ('a2','s2','projB','assistant','2026-04-11T00:00:01Z','claude-sonnet-4-6',5,5,0,0,0,NULL,NULL);
            INSERT INTO tool_calls (message_uuid, session_id, project_slug, tool_name, target, timestamp, is_error)
            VALUES ('a1','s1','projA','Read','foo.py','2026-04-10T00:00:01Z',0),
                   ('a1','s1','projA','Bash','npm test','2026-04-10T00:00:01Z',0);
            """)
            c.commit()

    def test_overview_totals(self):
        t = overview_totals(self.db, since=None, until=None)
        self.assertEqual(t["sessions"], 2)
        self.assertEqual(t["turns"], 2)
        self.assertEqual(t["input_tokens"], 105)
        self.assertEqual(t["output_tokens"], 205)

    def test_expensive_prompts_orders_by_assistant_tokens(self):
        rows = expensive_prompts(self.db, limit=10)
        self.assertGreaterEqual(len(rows), 2)
        self.assertEqual(rows[0]["prompt_text"], "big prompt")

    def test_project_summary_groups(self):
        rows = project_summary(self.db)
        slugs = {r["project_slug"]: r for r in rows}
        self.assertIn("projA", slugs)
        self.assertEqual(slugs["projA"]["turns"], 1)

    def test_tool_breakdown(self):
        rows = tool_token_breakdown(self.db)
        names = {r["tool_name"]: r for r in rows}
        self.assertIn("Read", names)
        self.assertIn("Bash", names)

    def test_recent_sessions(self):
        rows = recent_sessions(self.db, limit=5)
        self.assertEqual(rows[0]["session_id"], "s2")  # most recent

    def test_session_turns(self):
        rows = session_turns(self.db, "s1")
        self.assertEqual(len(rows), 2)
```

- [ ] **Step 2: Implement queries**

Append to `db.py`:
```python
def _range_clause(since, until):
    where, args = [], []
    if since:
        where.append("timestamp >= ?"); args.append(since)
    if until:
        where.append("timestamp < ?"); args.append(until)
    return ((" AND " + " AND ".join(where)) if where else "", args)

def overview_totals(db_path, since=None, until=None):
    rng, args = _range_clause(since, until)
    sql = f"""
      SELECT COUNT(DISTINCT session_id) AS sessions,
             SUM(CASE WHEN type='user' THEN 1 ELSE 0 END) AS turns,
             COALESCE(SUM(input_tokens),0)            AS input_tokens,
             COALESCE(SUM(output_tokens),0)           AS output_tokens,
             COALESCE(SUM(cache_read_tokens),0)       AS cache_read_tokens,
             COALESCE(SUM(cache_create_5m_tokens),0)  AS cache_create_5m_tokens,
             COALESCE(SUM(cache_create_1h_tokens),0)  AS cache_create_1h_tokens
        FROM messages WHERE 1=1 {rng}
    """
    with connect(db_path) as c:
        return dict(c.execute(sql, args).fetchone())

def expensive_prompts(db_path, limit=50):
    """Each user prompt joined with the immediately-following assistant turn's tokens."""
    sql = """
      SELECT u.uuid AS user_uuid, u.session_id, u.project_slug, u.timestamp,
             u.prompt_text, u.prompt_chars,
             a.uuid AS assistant_uuid, a.model,
             COALESCE(a.input_tokens,0)+COALESCE(a.output_tokens,0)
               +COALESCE(a.cache_create_5m_tokens,0)+COALESCE(a.cache_create_1h_tokens,0) AS billable_tokens,
             COALESCE(a.cache_read_tokens,0) AS cache_read_tokens
        FROM messages u
        JOIN messages a ON a.parent_uuid = u.uuid AND a.type='assistant'
       WHERE u.type='user' AND u.prompt_text IS NOT NULL
       ORDER BY billable_tokens DESC
       LIMIT ?
    """
    with connect(db_path) as c:
        return [dict(r) for r in c.execute(sql, (limit,))]

def project_summary(db_path):
    sql = """
      SELECT project_slug,
             COUNT(DISTINCT session_id) AS sessions,
             SUM(CASE WHEN type='user' THEN 1 ELSE 0 END) AS turns,
             SUM(input_tokens)+SUM(output_tokens)+SUM(cache_create_5m_tokens)+SUM(cache_create_1h_tokens) AS billable_tokens,
             SUM(cache_read_tokens) AS cache_read_tokens
        FROM messages
       GROUP BY project_slug
       ORDER BY billable_tokens DESC
    """
    with connect(db_path) as c:
        return [dict(r) for r in c.execute(sql)]

def tool_token_breakdown(db_path):
    sql = """
      SELECT tool_name,
             COUNT(*) AS calls,
             COALESCE(SUM(result_tokens),0) AS result_tokens
        FROM tool_calls
       WHERE tool_name != '_tool_result'
       GROUP BY tool_name
       ORDER BY result_tokens DESC, calls DESC
    """
    with connect(db_path) as c:
        return [dict(r) for r in c.execute(sql)]

def recent_sessions(db_path, limit=20):
    sql = """
      SELECT session_id, project_slug, MIN(timestamp) AS started, MAX(timestamp) AS ended,
             SUM(CASE WHEN type='user' THEN 1 ELSE 0 END) AS turns,
             SUM(input_tokens)+SUM(output_tokens) AS tokens
        FROM messages
       GROUP BY session_id
       ORDER BY ended DESC
       LIMIT ?
    """
    with connect(db_path) as c:
        return [dict(r) for r in c.execute(sql, (limit,))]

def session_turns(db_path, session_id):
    sql = """
      SELECT uuid, parent_uuid, type, timestamp, model, is_sidechain, agent_id,
             input_tokens, output_tokens, cache_read_tokens,
             cache_create_5m_tokens, cache_create_1h_tokens,
             prompt_text, prompt_chars, tool_calls_json
        FROM messages
       WHERE session_id = ?
       ORDER BY timestamp ASC
    """
    with connect(db_path) as c:
        return [dict(r) for r in c.execute(sql, (session_id,))]
```

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_queries -v`

- [ ] **Step 4: Commit**

```bash
git add token_dashboard/db.py tests/test_queries.py
git commit -m "feat(db): query helpers for overview, prompts, projects, tools, sessions"
```

---

## Task 10: tips.py — cache discipline rule

**Files:**
- Create: `token_dashboard/tips.py`
- Create: `tests/test_tips.py`

- [ ] **Step 1: Failing test**

```python
import os, tempfile, unittest
from token_dashboard.db import init_db, connect
from token_dashboard.tips import cache_discipline_tips

class CacheTipTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.mkdtemp()
        self.db = os.path.join(self.tmp, "t.db")
        init_db(self.db)

    def _ins(self, ts, project, cache_read, cache_create):
        with connect(self.db) as c:
            c.execute("""INSERT INTO messages (uuid, session_id, project_slug, type, timestamp,
                model, input_tokens, output_tokens, cache_read_tokens,
                cache_create_5m_tokens, cache_create_1h_tokens) VALUES
                (?, 's', ?, 'assistant', ?, 'claude-opus-4-7', 100, 100, ?, ?, 0)""",
                (f"uuid-{ts}", project, ts, cache_read, cache_create))
            c.commit()

    def test_low_cache_hit_emits_tip(self):
        self._ins("2026-04-15T00:00:00Z", "projX", 10, 1000)
        tips = cache_discipline_tips(self.db, today_iso="2026-04-19")
        self.assertTrue(any(t["category"] == "cache" for t in tips))

    def test_healthy_cache_no_tip(self):
        for i in range(10):
            self._ins(f"2026-04-15T00:00:0{i}Z", "projY", 1000, 50)
        tips = cache_discipline_tips(self.db, today_iso="2026-04-19")
        self.assertFalse(any(t["category"] == "cache" for t in tips))
```

- [ ] **Step 2: Implement**

```python
"""Rule-based tips engine — produces actionable suggestions from SQLite."""
from __future__ import annotations
from datetime import datetime, timedelta
from .db import connect

def _iso_days_ago(today_iso: str, n: int) -> str:
    d = datetime.fromisoformat(today_iso.replace("Z", ""))
    return (d - timedelta(days=n)).isoformat()

def _key(category: str, scope: str) -> str:
    return f"{category}:{scope}"

def _is_dismissed(db_path, key: str) -> bool:
    with connect(db_path) as c:
        r = c.execute("SELECT dismissed_at FROM dismissed_tips WHERE tip_key=?", (key,)).fetchone()
    if not r:
        return False
    # Re-show after 14 days
    import time
    return (time.time() - r["dismissed_at"]) < 14 * 86400

def cache_discipline_tips(db_path, today_iso: str | None = None) -> list[dict]:
    today_iso = today_iso or datetime.utcnow().isoformat()
    since = _iso_days_ago(today_iso, 7)
    sql = """
      SELECT project_slug,
             SUM(cache_read_tokens) AS cr,
             SUM(input_tokens + cache_create_5m_tokens + cache_create_1h_tokens) AS rebuild
        FROM messages
       WHERE type='assistant' AND timestamp >= ?
       GROUP BY project_slug
       HAVING (cr + rebuild) > 100000
    """
    out = []
    with connect(db_path) as c:
        for row in c.execute(sql, (since,)):
            total = (row["cr"] or 0) + (row["rebuild"] or 0)
            hit = (row["cr"] or 0) / total if total else 0
            if hit < 0.40:
                key = _key("cache", row["project_slug"])
                if _is_dismissed(db_path, key):
                    continue
                out.append({
                    "key": key,
                    "category": "cache",
                    "title": f"Low cache hit rate in {row['project_slug']}",
                    "body": f"Cache hit rate is {hit*100:.0f}% over the last 7 days. Sessions that restart context frequently rebuild cache. Consider longer-lived sessions or fewer context resets.",
                    "scope": row["project_slug"],
                })
    return out

def dismiss_tip(db_path, key: str) -> None:
    import time
    with connect(db_path) as c:
        c.execute("INSERT OR REPLACE INTO dismissed_tips (tip_key, dismissed_at) VALUES (?, ?)", (key, time.time()))
        c.commit()
```

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_tips -v`

- [ ] **Step 4: Commit**

```bash
git add token_dashboard/tips.py tests/test_tips.py
git commit -m "feat(tips): cache discipline rule with dismissal tracking"
```

---

## Task 11: tips.py — repeated reads + bash rule

**Files:**
- Modify: `token_dashboard/tips.py`
- Modify: `tests/test_tips.py`

- [ ] **Step 1: Failing test**

```python
class RepeatTipTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.mkdtemp()
        self.db = os.path.join(self.tmp, "t.db")
        init_db(self.db)
        with connect(self.db) as c:
            c.execute("INSERT INTO messages (uuid, session_id, project_slug, type, timestamp, model) VALUES ('m1','s1','p','assistant','2026-04-15T00:00:00Z','claude-opus-4-7')")
            for i in range(15):
                c.execute("INSERT INTO tool_calls (message_uuid, session_id, project_slug, tool_name, target, timestamp, is_error) VALUES ('m1','s1','p','Read','src/Root.tsx','2026-04-15T00:00:00Z',0)")
            for i in range(20):
                c.execute("INSERT INTO tool_calls (message_uuid, session_id, project_slug, tool_name, target, timestamp, is_error) VALUES ('m1','s1','p','Bash','npm run lint','2026-04-15T00:00:00Z',0)")
            c.commit()

    def test_repeated_file_read_tip(self):
        from token_dashboard.tips import repeated_target_tips
        tips = repeated_target_tips(self.db, today_iso="2026-04-19")
        cats = [t["category"] for t in tips]
        self.assertIn("repeat-file", cats)
        self.assertIn("repeat-bash", cats)
```

- [ ] **Step 2: Implement**

Append to `tips.py`:
```python
def repeated_target_tips(db_path, today_iso: str | None = None) -> list[dict]:
    today_iso = today_iso or datetime.utcnow().isoformat()
    since = _iso_days_ago(today_iso, 7)
    out = []
    with connect(db_path) as c:
        for row in c.execute("""
          SELECT target, COUNT(*) AS n, COUNT(DISTINCT session_id) AS sessions
            FROM tool_calls WHERE tool_name IN ('Read','Edit','Write') AND timestamp >= ?
            GROUP BY target HAVING n > 10 ORDER BY n DESC LIMIT 10
        """, (since,)):
            key = _key("repeat-file", row["target"] or "?")
            if _is_dismissed(db_path, key): continue
            out.append({
                "key": key, "category": "repeat-file",
                "title": f"{row['target']} read {row['n']} times",
                "body": f"This file was opened {row['n']} times across {row['sessions']} sessions in the past 7 days. Adding a summary to CLAUDE.md or referencing it once per session can avoid repeated reads.",
                "scope": row["target"],
            })
        for row in c.execute("""
          SELECT target, COUNT(*) AS n
            FROM tool_calls WHERE tool_name='Bash' AND timestamp >= ?
            GROUP BY target HAVING n > 15 ORDER BY n DESC LIMIT 10
        """, (since,)):
            key = _key("repeat-bash", row["target"] or "?")
            if _is_dismissed(db_path, key): continue
            out.append({
                "key": key, "category": "repeat-bash",
                "title": f"`{row['target']}` ran {row['n']} times",
                "body": f"This bash command was run {row['n']} times in the past 7 days. Consider a watch flag or a shell alias.",
                "scope": row["target"],
            })
    return out
```

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_tips -v`

- [ ] **Step 4: Commit**

```bash
git add token_dashboard/tips.py tests/test_tips.py
git commit -m "feat(tips): repeated file reads + bash command rule"
```

---

## Task 12: tips.py — model right-sizing rule

**Files:**
- Modify: `token_dashboard/tips.py`
- Modify: `tests/test_tips.py`

- [ ] **Step 1: Failing test**

```python
class RightSizeTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.mkdtemp(); self.db = os.path.join(self.tmp,"t.db"); init_db(self.db)

    def test_short_opus_turns_flagged(self):
        with connect(self.db) as c:
            for i in range(10):
                c.execute("INSERT INTO messages (uuid, session_id, project_slug, type, timestamp, model, input_tokens, output_tokens, cache_read_tokens, cache_create_5m_tokens, cache_create_1h_tokens) VALUES (?, 's','p','assistant','2026-04-18T00:00:00Z','claude-opus-4-7', 1000, 200, 0, 0, 0)", (f"a{i}",))
            c.commit()
        from token_dashboard.tips import right_size_tips
        tips = right_size_tips(self.db, today_iso="2026-04-19")
        self.assertTrue(any(t["category"] == "right-size" for t in tips))
```

- [ ] **Step 2: Implement**

Append to `tips.py`:
```python
def right_size_tips(db_path, today_iso: str | None = None) -> list[dict]:
    today_iso = today_iso or datetime.utcnow().isoformat()
    since = _iso_days_ago(today_iso, 7)
    sql = """
      SELECT COUNT(*) AS n,
             SUM(input_tokens+cache_create_5m_tokens+cache_create_1h_tokens) AS in_tok,
             SUM(output_tokens) AS out_tok
        FROM messages
       WHERE type='assistant' AND model LIKE '%opus%'
         AND output_tokens < 500 AND is_sidechain = 0
         AND timestamp >= ?
    """
    with connect(db_path) as c:
        row = c.execute(sql, (since,)).fetchone()
    if not row or (row["n"] or 0) < 10:
        return []
    # Estimate Sonnet vs Opus diff: opus_in/sonnet_in = 5x, opus_out/sonnet_out = 5x  (15→3, 75→15)
    api_opus = ((row["in_tok"] or 0) * 15 + (row["out_tok"] or 0) * 75) / 1_000_000
    api_sonnet = ((row["in_tok"] or 0) * 3 + (row["out_tok"] or 0) * 15) / 1_000_000
    savings = api_opus - api_sonnet
    if savings < 1.0:
        return []
    key = _key("right-size", "opus-short-turns-7d")
    if _is_dismissed(db_path, key): return []
    return [{
        "key": key, "category": "right-size",
        "title": f"{row['n']} short Opus turns might fit on Sonnet",
        "body": f"Opus turns under 500 output tokens accounted for ~${api_opus:.2f} in the last 7 days. Running them on Sonnet would have cost ~${api_sonnet:.2f} (savings ~${savings:.2f}).",
        "scope": "opus-short-turns-7d",
    }]
```

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_tips -v`

- [ ] **Step 4: Commit**

```bash
git add token_dashboard/tips.py tests/test_tips.py
git commit -m "feat(tips): model right-sizing rule for short Opus turns"
```

---

## Task 13: tips.py — outlier detection + aggregator

**Files:**
- Modify: `token_dashboard/tips.py`
- Modify: `tests/test_tips.py`

- [ ] **Step 1: Failing test**

```python
class OutlierTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.mkdtemp(); self.db = os.path.join(self.tmp,"t.db"); init_db(self.db)

    def test_giant_tool_result_flagged(self):
        with connect(self.db) as c:
            for i in range(20):
                c.execute("INSERT INTO messages (uuid, session_id, project_slug, type, timestamp) VALUES (?, 's','p','user','2026-04-18T00:00:00Z')", (f"u{i}",))
                c.execute("INSERT INTO tool_calls (message_uuid, session_id, project_slug, tool_name, target, result_tokens, timestamp, is_error) VALUES (?, 's','p','_tool_result','tu','100000','2026-04-18T00:00:00Z',0)", (f"u{i}",))
            c.commit()
        from token_dashboard.tips import outlier_tips
        tips = outlier_tips(self.db, today_iso="2026-04-19")
        self.assertTrue(any(t["category"] == "tool-bloat" for t in tips))
```

- [ ] **Step 2: Implement**

Append to `tips.py`:
```python
def outlier_tips(db_path, today_iso: str | None = None) -> list[dict]:
    today_iso = today_iso or datetime.utcnow().isoformat()
    since = _iso_days_ago(today_iso, 7)
    out = []
    with connect(db_path) as c:
        # large tool_results
        big = c.execute("""
          SELECT COUNT(*) AS n, AVG(result_tokens) AS avg_t
            FROM tool_calls WHERE tool_name='_tool_result' AND result_tokens > 50000 AND timestamp >= ?
        """, (since,)).fetchone()
        if big and (big["n"] or 0) >= 5:
            key = _key("tool-bloat", "result-50k+")
            if not _is_dismissed(db_path, key):
                out.append({
                    "key": key, "category": "tool-bloat",
                    "title": f"{big['n']} tool results over 50k tokens this week",
                    "body": f"Average size of these results is {int(big['avg_t']):,} tokens. Pipe long Bash output to head/tail and ask for narrower file reads.",
                    "scope": "result-50k+",
                })
        # subagent cost outliers (per agent_id)
        for row in c.execute("""
          SELECT agent_id, COUNT(*) AS n,
                 AVG(input_tokens+output_tokens) AS mean_t,
                 MAX(input_tokens+output_tokens) AS max_t
            FROM messages
           WHERE is_sidechain=1 AND agent_id IS NOT NULL AND timestamp >= ?
           GROUP BY agent_id HAVING n >= 5
        """, (since,)):
            if (row["max_t"] or 0) > 4 * (row["mean_t"] or 1):
                key = _key("subagent-outlier", row["agent_id"])
                if _is_dismissed(db_path, key): continue
                out.append({
                    "key": key, "category": "subagent-outlier",
                    "title": f"Subagent {row['agent_id']} has cost outliers",
                    "body": f"Largest invocation used {int(row['max_t']):,} tokens vs mean {int(row['mean_t']):,}. Worth checking what those did differently.",
                    "scope": row["agent_id"],
                })
    return out

def all_tips(db_path, today_iso: str | None = None) -> list[dict]:
    return [
        *cache_discipline_tips(db_path, today_iso),
        *repeated_target_tips(db_path, today_iso),
        *right_size_tips(db_path, today_iso),
        *outlier_tips(db_path, today_iso),
    ]
```

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_tips -v`

- [ ] **Step 4: Commit**

```bash
git add token_dashboard/tips.py tests/test_tips.py
git commit -m "feat(tips): outlier detection + aggregator (all_tips)"
```

---

## Task 14: cli.py — scan / today / stats / tips

**Files:**
- Create: `cli.py`
- Create: `tests/test_cli.py`

- [ ] **Step 1: Failing test**

```python
import io, os, subprocess, sys, tempfile, unittest
from token_dashboard.db import init_db

class CliTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.mkdtemp()
        self.proj = os.path.join(self.tmp, "projects")
        os.makedirs(os.path.join(self.proj, "demo"))
        with open(os.path.join(self.proj, "demo", "s.jsonl"), "w") as f:
            f.write('{"type":"user","uuid":"u1","sessionId":"s1","timestamp":"2026-04-19T00:00:00Z","isSidechain":false,"message":{"role":"user","content":"hi"}}\n')
            f.write('{"type":"assistant","uuid":"a1","parentUuid":"u1","sessionId":"s1","timestamp":"2026-04-19T00:00:01Z","isSidechain":false,"message":{"model":"claude-haiku-4-5","usage":{"input_tokens":1,"output_tokens":1}}}\n')
        self.db = os.path.join(self.tmp, "t.db")

    def _run(self, *args):
        env = {**os.environ, "TOKEN_DASHBOARD_DB": self.db}
        return subprocess.run(
            [sys.executable, "cli.py", *args],
            cwd=os.path.dirname(os.path.dirname(os.path.abspath(__file__))),
            env=env, capture_output=True, text=True,
        )

    def test_scan_then_today_text_output(self):
        r1 = self._run("scan", "--projects-dir", self.proj)
        self.assertEqual(r1.returncode, 0, r1.stderr)
        r2 = self._run("today")
        self.assertEqual(r2.returncode, 0, r2.stderr)
        # Today's date is the fixture date — but `today` uses real today, so just check it ran.
        self.assertIn("Token Dashboard", r2.stdout + r2.stderr)
```

- [ ] **Step 2: Implement cli.py**

```python
"""Token Dashboard CLI entrypoint."""
from __future__ import annotations
import argparse, os, sys, webbrowser
from datetime import datetime, timedelta, timezone
from pathlib import Path
from token_dashboard.db import init_db, default_db_path
from token_dashboard.scanner import scan_dir
from token_dashboard.pricing import load_pricing, cost_for, get_plan
from token_dashboard.tips import all_tips
from token_dashboard.db import overview_totals

def _db_path(args):
    return args.db or os.environ.get("TOKEN_DASHBOARD_DB") or str(default_db_path())

def _projects(args):
    return args.projects_dir or os.environ.get("CLAUDE_PROJECTS_DIR") or str(Path.home() / ".claude" / "projects")

def _today_range():
    now = datetime.now(timezone.utc)
    start = datetime(now.year, now.month, now.day, tzinfo=timezone.utc).isoformat()
    end = (now + timedelta(days=1)).replace(hour=0, minute=0, second=0, microsecond=0).isoformat()
    return start, end

def cmd_scan(args):
    db = _db_path(args)
    init_db(db)
    n = scan_dir(_projects(args), db)
    print(f"Token Dashboard: scanned {n['files']} files, {n['messages']} messages, {n['tools']} tool calls")

def cmd_today(args):
    db = _db_path(args); init_db(db)
    s, e = _today_range()
    t = overview_totals(db, since=s, until=e)
    print("Token Dashboard — today")
    print(f"  sessions: {t['sessions']}    turns: {t['turns']}")
    print(f"  input:    {t['input_tokens']:>12,}    output: {t['output_tokens']:>12,}")
    print(f"  cache rd: {t['cache_read_tokens']:>12,}    cache cr: {t['cache_create_5m_tokens']+t['cache_create_1h_tokens']:>12,}")

def cmd_stats(args):
    db = _db_path(args); init_db(db)
    t = overview_totals(db)
    print("Token Dashboard — all time")
    print(f"  sessions: {t['sessions']}    turns: {t['turns']}")
    print(f"  input:    {t['input_tokens']:>12,}    output: {t['output_tokens']:>12,}")

def cmd_tips(args):
    db = _db_path(args); init_db(db)
    for tip in all_tips(db):
        print(f"[{tip['category']}] {tip['title']}")
        print(f"  {tip['body']}\n")

def cmd_dashboard(args):
    db = _db_path(args); init_db(db)
    if not args.no_scan:
        scan_dir(_projects(args), db)
    from token_dashboard.server import run
    host = os.environ.get("HOST", "127.0.0.1")
    port = int(os.environ.get("PORT", "8080"))
    url = f"http://{host}:{port}/"
    if not args.no_open:
        webbrowser.open(url)
    print(f"Token Dashboard listening on {url}")
    run(host, port, db, _projects(args))

def main():
    p = argparse.ArgumentParser(prog="token-dashboard", description="Local Claude Code usage dashboard")
    p.add_argument("--db", help="SQLite path (default ~/.claude/token-dashboard.db)")
    p.add_argument("--projects-dir", help="JSONL root (default ~/.claude/projects)")
    sub = p.add_subparsers(dest="cmd", required=True)
    sub.add_parser("scan").set_defaults(func=cmd_scan)
    sub.add_parser("today").set_defaults(func=cmd_today)
    sub.add_parser("stats").set_defaults(func=cmd_stats)
    sub.add_parser("tips").set_defaults(func=cmd_tips)
    d = sub.add_parser("dashboard")
    d.add_argument("--no-scan", action="store_true")
    d.add_argument("--no-open", action="store_true")
    d.set_defaults(func=cmd_dashboard)
    args = p.parse_args()
    args.func(args)

if __name__ == "__main__":
    main()
```

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_cli -v`

- [ ] **Step 4: Commit**

```bash
git add cli.py tests/test_cli.py
git commit -m "feat(cli): scan/today/stats/tips/dashboard subcommands"
```

---

## Task 15: server.py — static + JSON endpoints

**Files:**
- Create: `token_dashboard/server.py`
- Create: `tests/test_server.py`

- [ ] **Step 1: Failing test**

```python
import json, os, sqlite3, tempfile, threading, unittest, urllib.request
from token_dashboard.db import init_db
from token_dashboard.server import build_handler
import http.server, socket

def _free_port():
    s = socket.socket(); s.bind(("127.0.0.1", 0)); p = s.getsockname()[1]; s.close(); return p

class ServerTests(unittest.TestCase):
    def setUp(self):
        self.tmp = tempfile.mkdtemp()
        self.db = os.path.join(self.tmp, "t.db")
        init_db(self.db)
        with sqlite3.connect(self.db) as c:
            c.execute("INSERT INTO messages (uuid, session_id, project_slug, type, timestamp, model, input_tokens, output_tokens, cache_read_tokens, cache_create_5m_tokens, cache_create_1h_tokens, prompt_text) VALUES ('u','s','p','user','2026-04-19T00:00:00Z',NULL,0,0,0,0,0,'hi')")
            c.execute("INSERT INTO messages (uuid, parent_uuid, session_id, project_slug, type, timestamp, model, input_tokens, output_tokens, cache_read_tokens, cache_create_5m_tokens, cache_create_1h_tokens) VALUES ('a','u','s','p','assistant','2026-04-19T00:00:01Z','claude-haiku-4-5',1,1,0,0,0)")
            c.commit()
        self.port = _free_port()
        H = build_handler(self.db, projects_dir="/nonexistent")
        self.httpd = http.server.HTTPServer(("127.0.0.1", self.port), H)
        threading.Thread(target=self.httpd.serve_forever, daemon=True).start()

    def tearDown(self):
        self.httpd.shutdown()

    def _get(self, path):
        return urllib.request.urlopen(f"http://127.0.0.1:{self.port}{path}").read()

    def test_index_html(self):
        body = self._get("/")
        self.assertIn(b"Token Dashboard", body)

    def test_overview_json(self):
        body = json.loads(self._get("/api/overview"))
        self.assertIn("sessions", body)
        self.assertEqual(body["sessions"], 1)

    def test_prompts_json(self):
        body = json.loads(self._get("/api/prompts?limit=10"))
        self.assertIsInstance(body, list)
```

- [ ] **Step 2: Implement server.py**

```python
"""HTTP server: static frontend + JSON endpoints + SSE diff stream."""
from __future__ import annotations
import http.server, json, mimetypes, os, queue, threading, time
from pathlib import Path
from urllib.parse import urlparse, parse_qs
from .db import (
    overview_totals, expensive_prompts, project_summary,
    tool_token_breakdown, recent_sessions, session_turns,
)
from .pricing import load_pricing, cost_for, get_plan, set_plan, format_for_user
from .tips import all_tips, dismiss_tip
from .scanner import scan_dir

WEB_ROOT = Path(__file__).resolve().parent.parent / "web"
PRICING_JSON = Path(__file__).resolve().parent.parent / "pricing.json"

def _send_json(self, obj, status=200):
    body = json.dumps(obj).encode("utf-8")
    self.send_response(status)
    self.send_header("Content-Type", "application/json")
    self.send_header("Content-Length", str(len(body)))
    self.send_header("Cache-Control", "no-store")
    self.end_headers()
    self.wfile.write(body)

def _serve_static(self, rel: str):
    p = (WEB_ROOT / rel.lstrip("/")).resolve()
    if not str(p).startswith(str(WEB_ROOT.resolve())) or not p.is_file():
        self.send_response(404); self.end_headers(); return
    body = p.read_bytes()
    ctype, _ = mimetypes.guess_type(str(p))
    self.send_response(200)
    self.send_header("Content-Type", ctype or "application/octet-stream")
    self.send_header("Content-Length", str(len(body)))
    self.end_headers(); self.wfile.write(body)

def build_handler(db_path: str, projects_dir: str):
    pricing = load_pricing(PRICING_JSON)

    class H(http.server.BaseHTTPRequestHandler):
        def log_message(self, fmt, *args): pass  # quiet

        def do_GET(self):
            url = urlparse(self.path)
            qs = parse_qs(url.query or "")
            path = url.path
            if path == "/" or path == "/index.html":
                return _serve_static(self, "index.html")
            if path.startswith("/web/") or path in ("/style.css", "/app.js", "/charts.js") or path.startswith("/routes/"):
                return _serve_static(self, path[5:] if path.startswith("/web/") else path)
            if path == "/api/overview":
                return _send_json(self, overview_totals(db_path, qs.get("since", [None])[0], qs.get("until", [None])[0]))
            if path == "/api/prompts":
                limit = int(qs.get("limit", ["50"])[0])
                rows = expensive_prompts(db_path, limit=limit)
                for r in rows:
                    c = cost_for(r["model"], {
                        "input_tokens": 0, "output_tokens": 0, "cache_read_tokens": r["cache_read_tokens"],
                        "cache_create_5m_tokens": 0, "cache_create_1h_tokens": 0,
                    }, pricing)
                    r["estimated_cost_usd"] = c["usd"]
                return _send_json(self, rows)
            if path == "/api/projects":
                return _send_json(self, project_summary(db_path))
            if path == "/api/tools":
                return _send_json(self, tool_token_breakdown(db_path))
            if path == "/api/sessions":
                return _send_json(self, recent_sessions(db_path, limit=int(qs.get("limit", ["20"])[0])))
            if path.startswith("/api/sessions/"):
                sid = path.rsplit("/", 1)[1]
                return _send_json(self, session_turns(db_path, sid))
            if path == "/api/tips":
                return _send_json(self, all_tips(db_path))
            if path == "/api/plan":
                return _send_json(self, {"plan": get_plan(db_path), "pricing": pricing})
            if path == "/api/scan":
                n = scan_dir(projects_dir, db_path)
                return _send_json(self, n)
            self.send_response(404); self.end_headers()

        def do_POST(self):
            url = urlparse(self.path)
            length = int(self.headers.get("Content-Length") or 0)
            body = json.loads(self.rfile.read(length) or b"{}") if length else {}
            if url.path == "/api/plan":
                set_plan(db_path, body.get("plan", "api"))
                return _send_json(self, {"ok": True})
            if url.path == "/api/tips/dismiss":
                dismiss_tip(db_path, body.get("key", ""))
                return _send_json(self, {"ok": True})
            self.send_response(404); self.end_headers()

    return H

def run(host: str, port: int, db_path: str, projects_dir: str):
    H = build_handler(db_path, projects_dir)
    httpd = http.server.ThreadingHTTPServer((host, port), H)
    httpd.serve_forever()
```

> **Note:** Static `index.html` doesn't exist yet — the test only checks for "Token Dashboard" in body. We'll write a minimal `web/index.html` in this same task to make the test pass.

Create `web/index.html`:
```html
<!doctype html>
<html lang="en"><head>
<meta charset="utf-8"><title>Token Dashboard</title>
<link rel="stylesheet" href="/web/style.css">
</head><body>
<header><h1>Token Dashboard</h1></header>
<main id="app">Loading…</main>
<script type="module" src="/web/app.js"></script>
</body></html>
```

Create empty `web/style.css` and `web/app.js` (placeholders — populated next tasks):
```bash
touch web/style.css web/app.js
```

- [ ] **Step 3: Run tests, expect PASS**

Run: `python -m unittest tests.test_server -v`

- [ ] **Step 4: Commit**

```bash
git add token_dashboard/server.py web/index.html web/style.css web/app.js tests/test_server.py
git commit -m "feat(server): http.server with static + JSON endpoints"
```

---

## Task 16: server.py — SSE diff stream

**Files:**
- Modify: `token_dashboard/server.py`

- [ ] **Step 1: Add SSE plumbing**

In `server.py`, add a module-level `EVENTS: queue.Queue` and a periodic scan thread:

```python
EVENTS: "queue.Queue[dict]" = queue.Queue()

def _scan_loop(db_path: str, projects_dir: str, interval: float = 30.0):
    while True:
        try:
            n = scan_dir(projects_dir, db_path)
            if n["messages"] > 0:
                EVENTS.put({"type": "scan", "n": n, "ts": time.time()})
        except Exception as e:
            EVENTS.put({"type": "error", "message": str(e)})
        time.sleep(interval)
```

In `do_GET`, add `/api/stream`:
```python
            if path == "/api/stream":
                self.send_response(200)
                self.send_header("Content-Type", "text/event-stream")
                self.send_header("Cache-Control", "no-store")
                self.send_header("Connection", "keep-alive")
                self.end_headers()
                while True:
                    try:
                        evt = EVENTS.get(timeout=15)
                        self.wfile.write(f"data: {json.dumps(evt)}\n\n".encode())
                    except queue.Empty:
                        self.wfile.write(b": ping\n\n")
                    try:
                        self.wfile.flush()
                    except (BrokenPipeError, ConnectionResetError):
                        return
                return
```

In `run()`, start the scan thread:
```python
def run(host, port, db_path, projects_dir):
    threading.Thread(target=_scan_loop, args=(db_path, projects_dir), daemon=True).start()
    H = build_handler(db_path, projects_dir)
    httpd = http.server.ThreadingHTTPServer((host, port), H)
    httpd.serve_forever()
```

- [ ] **Step 2: Smoke test by hand (no automated test for SSE)**

```bash
python cli.py dashboard --no-open --no-scan &
sleep 1
curl -N --max-time 3 http://127.0.0.1:8080/api/stream | head -3
kill %1
```

Expected: at least one `: ping` line.

- [ ] **Step 3: Commit**

```bash
git add token_dashboard/server.py
git commit -m "feat(server): SSE event stream + 30s rescan loop"
```

---

## Task 17: web/style.css — base theme

**Files:**
- Modify: `web/style.css`

- [ ] **Step 1: Write the theme**

```css
:root {
  --bg:        #0A0E14;
  --panel:    #0F1419;
  --panel-2: #131922;
  --border:  #1F2630;
  --text:    #E6EDF3;
  --muted:   #8B98A6;
  --accent:  #4A9EFF;
  --accent-2:#7C5CFF;
  --good:    #3FB68B;
  --warn:    #E8A23B;
  --bad:     #E5484D;
  --mono: 'JetBrains Mono', ui-monospace, SFMono-Regular, Consolas, monospace;
  --sans: 'Inter', system-ui, -apple-system, Segoe UI, sans-serif;
}
* { box-sizing: border-box; }
html, body { background: var(--bg); color: var(--text); margin: 0; font-family: var(--sans); font-size: 13px; line-height: 1.5; }
a { color: var(--accent); text-decoration: none; }
a:hover { text-decoration: underline; }

header.topbar {
  display: flex; align-items: center; gap: 16px;
  padding: 10px 20px; border-bottom: 1px solid var(--border);
  background: var(--panel);
}
.topbar .brand { font-weight: 600; letter-spacing: -0.01em; }
.topbar nav { display: flex; gap: 4px; margin-left: auto; }
.topbar nav a { padding: 4px 10px; border-radius: 6px; color: var(--muted); }
.topbar nav a.active, .topbar nav a:hover { color: var(--text); background: var(--panel-2); text-decoration: none; }

main#app { padding: 20px; max-width: 1400px; margin: 0 auto; }

.card { background: var(--panel); border: 1px solid var(--border); border-radius: 8px; padding: 16px; }
.row { display: grid; gap: 16px; }
.row.cols-4 { grid-template-columns: repeat(4, 1fr); }
.row.cols-3 { grid-template-columns: repeat(3, 1fr); }
.row.cols-2 { grid-template-columns: 1fr 1fr; }

.kpi .value { font-family: var(--mono); font-size: 28px; font-variant-numeric: tabular-nums; letter-spacing: -0.02em; }
.kpi .label { color: var(--muted); text-transform: uppercase; font-size: 11px; letter-spacing: 0.06em; }
.kpi .delta { font-family: var(--mono); font-size: 12px; }
.kpi .delta.up   { color: var(--good); }
.kpi .delta.down { color: var(--bad); }

table { width: 100%; border-collapse: collapse; font-size: 13px; }
th, td { text-align: left; padding: 8px 12px; border-bottom: 1px solid var(--border); }
th { color: var(--muted); font-weight: 500; text-transform: uppercase; font-size: 11px; letter-spacing: 0.06em; }
td.num, th.num { font-family: var(--mono); text-align: right; font-variant-numeric: tabular-nums; }

.badge { display: inline-block; padding: 1px 6px; border-radius: 4px; font-size: 11px; background: var(--panel-2); color: var(--muted); border: 1px solid var(--border); }
.badge.opus    { color: var(--accent-2); border-color: rgba(124,92,255,0.3); }
.badge.sonnet  { color: var(--accent);   border-color: rgba(74,158,255,0.3); }
.badge.haiku   { color: var(--good);     border-color: rgba(63,182,139,0.3); }

.blur-sensitive.privacy-on { filter: blur(6px); }
```

Append to `web/index.html` `<head>`:
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
<script src="https://cdn.jsdelivr.net/npm/echarts@5.5.0/dist/echarts.min.js"></script>
```

- [ ] **Step 2: Smoke check**

```bash
python cli.py dashboard --no-open --no-scan &
sleep 1
curl -s http://127.0.0.1:8080/web/style.css | head -5
kill %1
```

Expected: CSS variables visible.

- [ ] **Step 3: Commit**

```bash
git add web/style.css web/index.html
git commit -m "feat(ui): dark theme tokens + Inter/JetBrains Mono + ECharts CDN"
```

---

## Task 18: web/app.js — router + fetch helpers + state

**Files:**
- Modify: `web/app.js`

- [ ] **Step 1: Implement**

```javascript
// app.js — router, state, fetch helpers
const $ = (sel, root=document) => root.querySelector(sel);
const $$ = (sel, root=document) => Array.from(root.querySelectorAll(sel));
export const fmt = {
  int: n => (n ?? 0).toLocaleString(),
  usd: n => n == null ? '—' : '$' + n.toFixed(2),
  pct: n => n == null ? '—' : (n * 100).toFixed(0) + '%',
  short: (s, n=80) => s == null ? '' : (s.length > n ? s.slice(0, n - 1) + '…' : s),
};
export async function api(path, opts) {
  const r = await fetch(path, opts);
  if (!r.ok) throw new Error(`${path} → ${r.status}`);
  return r.json();
}
export const state = { plan: 'api', pricing: null, range: '7d' };

const ROUTES = {
  '/overview': () => import('/web/routes/overview.js'),
  '/prompts':  () => import('/web/routes/prompts.js'),
  '/sessions': () => import('/web/routes/sessions.js'),
  '/tools':    () => import('/web/routes/tools.js'),
  '/projects': () => import('/web/routes/projects.js'),
  '/tips':     () => import('/web/routes/tips.js'),
  '/settings': () => import('/web/routes/settings.js'),
};

function buildTopbar() {
  const wrap = document.createElement('header');
  wrap.className = 'topbar';
  wrap.innerHTML = `
    <div class="brand">Token Dashboard</div>
    <nav>
      ${Object.keys(ROUTES).map(p => `<a href="#${p}" data-route="${p}">${p.slice(1)}</a>`).join('')}
    </nav>`;
  document.body.prepend(wrap);
}

async function render() {
  let path = location.hash.replace(/^#/, '') || '/overview';
  // Match /sessions/:id
  if (path.startsWith('/sessions/')) path = '/sessions';
  const loader = ROUTES[path] || ROUTES['/overview'];
  $$('header.topbar nav a').forEach(a => a.classList.toggle('active', a.dataset.route === path));
  const mod = await loader();
  $('#app').innerHTML = '';
  await mod.default($('#app'));
}

async function boot() {
  buildTopbar();
  state.plan = (await api('/api/plan')).plan;
  state.pricing = (await api('/api/plan')).pricing;
  window.addEventListener('hashchange', render);
  await render();
  // Privacy blur (Cmd+B / Ctrl+B)
  window.addEventListener('keydown', e => {
    if ((e.metaKey || e.ctrlKey) && e.key.toLowerCase() === 'b') {
      e.preventDefault();
      document.body.classList.toggle('privacy-on');
    }
  });
  // SSE
  const es = new EventSource('/api/stream');
  es.onmessage = ev => {
    try {
      const evt = JSON.parse(ev.data);
      if (evt.type === 'scan') render();
    } catch {}
  };
}

boot();
```

- [ ] **Step 2: Commit**

```bash
git add web/app.js
git commit -m "feat(ui): hash router + fetch helpers + privacy blur + SSE refresh"
```

---

## Task 19: web/charts.js — ECharts wrappers

**Files:**
- Create: `web/charts.js`

- [ ] **Step 1: Implement**

```javascript
const THEME = {
  textStyle: { color: '#E6EDF3', fontFamily: 'Inter' },
  grid: { left: 36, right: 12, top: 24, bottom: 24, containLabel: true },
  xAxis: { axisLine: { lineStyle: { color: '#1F2630' } }, axisLabel: { color: '#8B98A6' } },
  yAxis: { axisLine: { show: false }, splitLine: { lineStyle: { color: '#1F2630' } }, axisLabel: { color: '#8B98A6' } },
};

function mount(el) {
  const c = echarts.init(el, null, { renderer: 'svg' });
  window.addEventListener('resize', () => c.resize());
  return c;
}

export function lineChart(el, { x, series }) {
  const c = mount(el);
  c.setOption({
    ...THEME,
    tooltip: { trigger: 'axis', backgroundColor: '#0F1419', borderColor: '#1F2630', textStyle: { color: '#E6EDF3' } },
    legend: { textStyle: { color: '#8B98A6' }, top: 0, right: 0 },
    xAxis: { ...THEME.xAxis, type: 'category', data: x },
    yAxis: { ...THEME.yAxis, type: 'value' },
    series: series.map(s => ({ ...s, type: 'line', smooth: true, areaStyle: { opacity: 0.1 } })),
  });
  return c;
}

export function barChart(el, { categories, values, color = '#4A9EFF' }) {
  const c = mount(el);
  c.setOption({
    ...THEME,
    tooltip: { trigger: 'axis', backgroundColor: '#0F1419', borderColor: '#1F2630', textStyle: { color: '#E6EDF3' } },
    xAxis: { ...THEME.xAxis, type: 'category', data: categories },
    yAxis: { ...THEME.yAxis, type: 'value' },
    series: [{ type: 'bar', data: values, itemStyle: { color, borderRadius: [4,4,0,0] } }],
  });
  return c;
}

export function donutChart(el, data) {
  const c = mount(el);
  c.setOption({
    tooltip: { backgroundColor: '#0F1419', borderColor: '#1F2630', textStyle: { color: '#E6EDF3' } },
    legend: { textStyle: { color: '#8B98A6' }, bottom: 0 },
    series: [{
      type: 'pie', radius: ['55%', '80%'], avoidLabelOverlap: true,
      label: { show: false }, labelLine: { show: false },
      data,
    }],
  });
  return c;
}
```

- [ ] **Step 2: Commit**

```bash
git add web/charts.js
git commit -m "feat(ui): ECharts wrappers (line, bar, donut) themed for dark UI"
```

---

## Task 20: web/routes/overview.js

**Files:**
- Create: `web/routes/overview.js`

- [ ] **Step 1: Implement**

```javascript
import { api, fmt, state } from '/web/app.js';
import { lineChart, donutChart, barChart } from '/web/charts.js';

export default async function (root) {
  const totals = await api('/api/overview');
  const projects = await api('/api/projects');
  const sessions = await api('/api/sessions?limit=8');
  const tools = await api('/api/tools');

  const cacheTotal = totals.cache_read_tokens + totals.cache_create_5m_tokens + totals.cache_create_1h_tokens + totals.input_tokens;
  const cacheHit = cacheTotal ? totals.cache_read_tokens / cacheTotal : 0;

  root.innerHTML = `
    <div class="row cols-4">
      <div class="card kpi"><div class="label">Sessions</div><div class="value">${fmt.int(totals.sessions)}</div></div>
      <div class="card kpi"><div class="label">Turns</div><div class="value">${fmt.int(totals.turns)}</div></div>
      <div class="card kpi"><div class="label">Output tokens</div><div class="value">${fmt.int(totals.output_tokens)}</div></div>
      <div class="card kpi"><div class="label">Cache hit</div><div class="value">${fmt.pct(cacheHit)}</div></div>
    </div>
    <div class="row cols-2" style="margin-top:16px">
      <div class="card"><h3 style="margin:0 0 8px">Tokens by project</h3><div id="ch-projects" style="height:280px"></div></div>
      <div class="card"><h3 style="margin:0 0 8px">Top tools</h3><div id="ch-tools" style="height:280px"></div></div>
    </div>
    <div class="card" style="margin-top:16px">
      <h3 style="margin:0 0 8px">Recent sessions</h3>
      <table>
        <thead><tr><th>started</th><th>project</th><th class="num">turns</th><th class="num">tokens</th><th>session</th></tr></thead>
        <tbody>
          ${sessions.map(s => `<tr><td>${s.started?.slice(0,16).replace('T',' ') || ''}</td><td>${s.project_slug}</td><td class="num">${fmt.int(s.turns)}</td><td class="num">${fmt.int(s.tokens)}</td><td><a href="#/sessions/${s.session_id}">${s.session_id.slice(0,8)}…</a></td></tr>`).join('')}
        </tbody>
      </table>
    </div>
  `;
  barChart(document.getElementById('ch-projects'),
    { categories: projects.slice(0,8).map(p=>p.project_slug), values: projects.slice(0,8).map(p=>p.billable_tokens) });
  barChart(document.getElementById('ch-tools'),
    { categories: tools.slice(0,8).map(t=>t.tool_name), values: tools.slice(0,8).map(t=>t.calls), color: '#7C5CFF' });
}
```

- [ ] **Step 2: Open in browser, smoke check**

```bash
python cli.py dashboard
```

Verify: KPI cards render with real numbers, two charts render, recent sessions table populated.

- [ ] **Step 3: Commit**

```bash
git add web/routes/overview.js
git commit -m "feat(ui): overview page with KPIs, project/tool charts, recent sessions"
```

---

## Task 21: web/routes/prompts.js

**Files:**
- Create: `web/routes/prompts.js`

- [ ] **Step 1: Implement**

```javascript
import { api, fmt } from '/web/app.js';

export default async function (root) {
  const rows = await api('/api/prompts?limit=100');
  root.innerHTML = `
    <div class="card">
      <h2 style="margin:0 0 12px">Expensive prompts</h2>
      <table id="prompts">
        <thead><tr><th class="num">$</th><th>prompt</th><th>model</th><th class="num">tokens</th><th class="num">cache rd</th><th>session</th></tr></thead>
        <tbody>
          ${rows.map((r,i) => `
            <tr data-i="${i}">
              <td class="num">${fmt.usd(r.estimated_cost_usd)}</td>
              <td class="blur-sensitive">${fmt.short(r.prompt_text, 100)}</td>
              <td><span class="badge ${(r.model||'').includes('opus')?'opus':(r.model||'').includes('sonnet')?'sonnet':'haiku'}">${(r.model||'').replace('claude-','')}</span></td>
              <td class="num">${fmt.int(r.billable_tokens)}</td>
              <td class="num">${fmt.int(r.cache_read_tokens)}</td>
              <td><a href="#/sessions/${r.session_id}">${r.session_id.slice(0,8)}…</a></td>
            </tr>`).join('')}
        </tbody>
      </table>
    </div>
    <aside id="drawer" hidden></aside>
  `;
  root.querySelectorAll('#prompts tbody tr').forEach(tr => {
    tr.addEventListener('click', () => {
      const r = rows[Number(tr.dataset.i)];
      const d = document.getElementById('drawer');
      d.hidden = false;
      d.innerHTML = `<div class="card" style="margin-top:16px">
        <h3 style="margin:0 0 8px">Prompt detail</h3>
        <p class="blur-sensitive" style="font-family:var(--mono); white-space:pre-wrap">${(r.prompt_text||'').replace(/[<>]/g, c => c==='<'?'&lt;':'&gt;')}</p>
        <p><a href="#/sessions/${r.session_id}">Open session →</a></p>
      </div>`;
    });
  });
}
```

- [ ] **Step 2: Smoke check**

Visit `#/prompts`. Verify table sorts by cost descending; clicking a row opens the drawer.

- [ ] **Step 3: Commit**

```bash
git add web/routes/prompts.js
git commit -m "feat(ui): expensive-prompts leaderboard with click-to-expand drawer"
```

---

## Task 22: web/routes/sessions.js

**Files:**
- Create: `web/routes/sessions.js`

- [ ] **Step 1: Implement**

```javascript
import { api, fmt } from '/web/app.js';

export default async function (root) {
  const id = location.hash.split('/')[2];
  if (!id) {
    const list = await api('/api/sessions?limit=50');
    root.innerHTML = `<div class="card"><h2>Sessions</h2><table><thead><tr><th>started</th><th>project</th><th class="num">turns</th><th class="num">tokens</th></tr></thead><tbody>
      ${list.map(s => `<tr><td>${s.started?.slice(0,16).replace('T',' ')}</td><td>${s.project_slug}</td><td class="num">${fmt.int(s.turns)}</td><td class="num">${fmt.int(s.tokens)}</td></tr>`).join('')}
    </tbody></table></div>`;
    return;
  }
  const turns = await api('/api/sessions/' + encodeURIComponent(id));
  root.innerHTML = `<div class="card"><h2>Session ${id.slice(0,8)}…</h2>
    <table><thead><tr><th>time</th><th>type</th><th>model</th><th class="blur-sensitive">prompt / tool</th><th class="num">in</th><th class="num">out</th><th class="num">cache rd</th></tr></thead><tbody>
    ${turns.map(t => `<tr>
      <td>${t.timestamp?.slice(11,19) || ''}</td>
      <td>${t.type}${t.is_sidechain?' <span class="badge">side</span>':''}</td>
      <td>${t.model ? `<span class="badge ${(t.model||'').includes('opus')?'opus':(t.model||'').includes('sonnet')?'sonnet':'haiku'}">${t.model.replace('claude-','')}</span>` : ''}</td>
      <td class="blur-sensitive">${fmt.short(t.prompt_text || (t.tool_calls_json || ''), 100)}</td>
      <td class="num">${fmt.int(t.input_tokens)}</td>
      <td class="num">${fmt.int(t.output_tokens)}</td>
      <td class="num">${fmt.int(t.cache_read_tokens)}</td>
    </tr>`).join('')}
    </tbody></table></div>`;
}
```

- [ ] **Step 2: Smoke check**

Click a session from overview, verify per-turn table.

- [ ] **Step 3: Commit**

```bash
git add web/routes/sessions.js
git commit -m "feat(ui): sessions list + per-session turn timeline"
```

---

## Task 23: web/routes/tools.js + projects.js + tips.js + settings.js

**Files:**
- Create: `web/routes/tools.js`
- Create: `web/routes/projects.js`
- Create: `web/routes/tips.js`
- Create: `web/routes/settings.js`

- [ ] **Step 1: tools.js**

```javascript
import { api, fmt } from '/web/app.js';
import { barChart } from '/web/charts.js';

export default async function (root) {
  const tools = await api('/api/tools');
  root.innerHTML = `
    <div class="card"><h2>Tool & file heatmap</h2>
      <div id="ch" style="height:320px"></div>
      <table style="margin-top:16px"><thead><tr><th>tool</th><th class="num">calls</th><th class="num">est tokens (results)</th></tr></thead><tbody>
        ${tools.map(t => `<tr><td>${t.tool_name}</td><td class="num">${fmt.int(t.calls)}</td><td class="num">${fmt.int(t.result_tokens)}</td></tr>`).join('')}
      </tbody></table>
    </div>`;
  barChart(document.getElementById('ch'),
    { categories: tools.slice(0,12).map(t=>t.tool_name), values: tools.slice(0,12).map(t=>t.result_tokens), color: '#7C5CFF' });
}
```

- [ ] **Step 2: projects.js**

```javascript
import { api, fmt } from '/web/app.js';
export default async function (root) {
  const rows = await api('/api/projects');
  root.innerHTML = `<div class="card"><h2>Projects</h2>
    <table><thead><tr><th>project</th><th class="num">sessions</th><th class="num">turns</th><th class="num">billable tokens</th><th class="num">cache rd</th></tr></thead><tbody>
      ${rows.map(r => `<tr><td>${r.project_slug}</td><td class="num">${fmt.int(r.sessions)}</td><td class="num">${fmt.int(r.turns)}</td><td class="num">${fmt.int(r.billable_tokens)}</td><td class="num">${fmt.int(r.cache_read_tokens)}</td></tr>`).join('')}
    </tbody></table></div>`;
}
```

- [ ] **Step 3: tips.js**

```javascript
import { api } from '/web/app.js';
export default async function (root) {
  const tips = await api('/api/tips');
  root.innerHTML = `<div class="card"><h2>Suggestions</h2>
    ${tips.length === 0 ? '<p style="color:var(--muted)">No suggestions right now.</p>' : ''}
    ${tips.map(t => `
      <div style="border:1px solid var(--border); border-radius:6px; padding:12px; margin:8px 0; background:var(--panel-2)">
        <div style="display:flex; align-items:center; gap:8px"><span class="badge">${t.category}</span><strong>${t.title}</strong>
          <button data-key="${t.key}" style="margin-left:auto; background:transparent; color:var(--muted); border:1px solid var(--border); border-radius:4px; padding:2px 8px; cursor:pointer">dismiss</button></div>
        <p style="margin:8px 0 0; color:var(--muted)">${t.body}</p>
      </div>`).join('')}
  </div>`;
  root.querySelectorAll('button[data-key]').forEach(b => b.addEventListener('click', async () => {
    await fetch('/api/tips/dismiss', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ key: b.dataset.key }) });
    location.reload();
  }));
}
```

- [ ] **Step 4: settings.js**

```javascript
import { api, state } from '/web/app.js';
export default async function (root) {
  const cur = await api('/api/plan');
  const plans = Object.entries(cur.pricing.plans);
  root.innerHTML = `<div class="card"><h2>Plan</h2>
    <p style="color:var(--muted)">Sets how cost is displayed. API mode shows pay-per-token. Subscription modes show what you actually pay.</p>
    <select id="plan">${plans.map(([k,v]) => `<option value="${k}" ${k===cur.plan?'selected':''}>${v.label} ${v.monthly?`($${v.monthly}/mo)`:''}</option>`).join('')}</select>
    <button id="save" style="margin-left:8px">Save</button>
    <p id="msg" style="color:var(--good)"></p>
  </div>`;
  document.getElementById('save').addEventListener('click', async () => {
    const plan = document.getElementById('plan').value;
    await fetch('/api/plan', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ plan }) });
    state.plan = plan;
    document.getElementById('msg').textContent = 'Saved.';
  });
}
```

- [ ] **Step 5: Smoke check each**

Visit `#/tools`, `#/projects`, `#/tips`, `#/settings`. Each renders without console errors.

- [ ] **Step 6: Commit**

```bash
git add web/routes/tools.js web/routes/projects.js web/routes/tips.js web/routes/settings.js
git commit -m "feat(ui): tools heatmap, projects, tips, settings pages"
```

---

## Task 24: End-to-end smoke + first-run plan modal

**Files:**
- Modify: `web/app.js` (first-run modal)

- [ ] **Step 1: Add first-run check**

In `boot()` after fetching `state.pricing`:
```javascript
  // First-run plan picker
  if (!localStorage.getItem('td.plan-set')) {
    const plans = Object.entries(state.pricing.plans);
    const overlay = document.createElement('div');
    overlay.style.cssText = 'position:fixed;inset:0;background:rgba(10,14,20,0.85);display:grid;place-items:center;z-index:50';
    overlay.innerHTML = `<div class="card" style="max-width:440px">
      <h2 style="margin:0 0 8px">Pick your plan</h2>
      <p style="color:var(--muted)">This sets how costs are displayed. You can change it later in Settings.</p>
      <select id="firstplan">${plans.map(([k,v]) => `<option value="${k}">${v.label}</option>`).join('')}</select>
      <button id="firstsave" style="margin-left:8px">Continue</button>
    </div>`;
    document.body.appendChild(overlay);
    await new Promise(res => overlay.querySelector('#firstsave').addEventListener('click', async () => {
      const plan = overlay.querySelector('#firstplan').value;
      await fetch('/api/plan', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ plan }) });
      localStorage.setItem('td.plan-set', '1');
      overlay.remove(); res();
    }));
    state.plan = (await api('/api/plan')).plan;
  }
```

- [ ] **Step 2: Full E2E sanity**

```bash
rm -f ~/.claude/token-dashboard.db
python cli.py dashboard
```

Walk through:
1. First-run modal → choose plan
2. Overview KPIs render with real numbers
3. Click into a recent session → turns visible
4. `/prompts` → leaderboard sorted by cost; click → drawer opens
5. `/tools` → bar chart visible
6. `/projects` → table renders
7. `/tips` → at least one tip if you have heavy usage; dismiss works
8. Cmd+B blurs prompt text
9. Settings → swap plan, refresh, KPIs update

- [ ] **Step 3: Commit**

```bash
git add web/app.js
git commit -m "feat(ui): first-run plan picker + final E2E polish"
```

---

## Verification

- All unit tests pass: `python -m unittest discover tests -v`
- Cold scan against `~/.claude/projects/` (~730 MB, 2,555 files) completes in <2 min: `time python cli.py scan`
- Warm rescan completes in <500 ms: `time python cli.py scan` (second run)
- Dashboard renders all eight tabs against real data with no console errors
- Privacy blur (Cmd+B) hides every `.blur-sensitive` element
- Plan switch in Settings updates cost display without rescan
- Tip dismissal persists across server restart (check `dismissed_tips` table)

---

## Self-Review

- ✅ Spec coverage: all eight features mapped to tasks (KPIs/overview→T20, expensive prompts→T21, sessions→T22, tools→T23, projects→T23, tips→T13/T23, cache analytics→T20 KPI + T13 rule, subagent attribution→T22 sidechain badge + T13 outlier rule). Pricing & plans→T8/T15/T23 settings.
- ✅ No placeholders or "TBD".
- ✅ Type consistency: `usage` keys (`input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_create_5m_tokens`, `cache_create_1h_tokens`) consistent across scanner/db/pricing.
- ✅ Tip categories used in tests (`cache`, `repeat-file`, `repeat-bash`, `right-size`, `tool-bloat`, `subagent-outlier`) match what `all_tips` returns.
- ✅ Server route handlers match the URLs the routes/ files call.
