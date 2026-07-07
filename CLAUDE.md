# CLAUDE.md

## SESSION PROTOCOL — FOLLOW BEFORE DOING ANYTHING

**Read and follow `SESSION_RUNNER.md` step by step.** It is your operating procedure for every session. It tells you what to read, when to stop, and how to close out.

**Three rules you will be tempted to violate:**
1. **Orient first** — Read SAFEGUARDS.md → SESSION_NOTES.md → `gh issue list` → git status → report findings → WAIT FOR THE USER TO SPEAK
2. **1 and done** — One deliverable per session. When it's complete, close out. Do not start the next thing.
3. **Auto-close** — When done: evaluate previous handoff, self-assess, document learnings, write handoff notes, commit, report, STOP.

`SESSION_RUNNER.md` documents known failure modes and their countermeasures. The protocol compensates for documented tendencies to skip orientation, skip close-out, and continue past the deliverable.

## Project Overview

Ham Radio Olympics — a web application for ham radio competitions. Python/FastAPI backend with SQLite, deployed via Fly.io.

### Key Components
- `main.py` — FastAPI application entry point
- `database.py` — SQLite database layer
- `scoring.py` — Competition scoring logic
- `auth.py` — Authentication
- `notifications.py` — Push notifications (POTA spots, etc.)
- `templates/` — Jinja2 HTML templates
- `static/` — Frontend assets
- `tests/` — Test suite

### Run Tests
```bash
pytest
```

### Run Locally
```bash
python main.py
```

### Deploy
Deployed on Fly.io. See `fly.toml` for configuration.

## Backlog & Issues

Use **GitHub Issues** for backlog items, bugs, and cross-project coordination. Do not use BACKLOG.md — it has been migrated to GitHub Issues.

- View issues: `gh issue list`
- Create issue: `gh issue create --title "..." --body "..."`
- Cross-project issues: use the target repo's issue tracker

## Project-Specific Methodology Adaptations

### Project-specific Learnings

Project learnings recorded by sessions in this repo. Per methodology v2.6.1+ (Phase 3C), project learnings live here in `CLAUDE.md`, not in the synced `SESSION_RUNNER.md` Learnings table — that table is kept byte-identical to canonical so `methodology/bin/sync` can upgrade it. (Migrated here from the project's `SESSION_RUNNER.md` Learnings table during the v2.8 sync.)

| # | Learning | Source | When to Apply |
|---|----------|--------|---------------|
| 1 | "Update the methodology" means: fetch from GitHub (`KJ5HST/methodology`), diff all methodology files against the remote, and sync any changes. Don't ask for clarification — just do it. Use `gh api` to fetch files, not the local sibling repo. | Session 1–2 | When user says "update methodology" or similar — fetch from GitHub and diff. |
| 2 | To diagnose a live scoring/data bug, there is no local DB — the SQLite DB lives only in production (`kd5dx:/data/ham_olympics.db`). Query it **read-only** with the operator's OK: `base64` a Python script locally, then `fly ssh console -a kd5dx -C "/bin/sh -lc 'echo <B64> \| base64 -d \| python3'"`, opening the DB as `file:/data/ham_olympics.db?mode=ro`. A QSO scores only if it clears every gate in `scoring.py:get_matching_qsos` — **confirmed** (`is_confirmed=1` unless `match.show_live_results`), in a match window, opted into the sport, mode/power OK, and target-matches; park *activations* also need 10+ QSOs/day/park. The `pota_bonus` is flat per (match, role), not per QSO. | Session 37 (P2P diagnosis) | Any "my score didn't change" / scoring bug report — check confirmation status first, then target-match, before assuming a code bug. |
| 3 | "Passes locally, fails on CI" for a sqlite **table-rebuild migration** (`ALTER TABLE x RENAME TO x_old` → `CREATE TABLE x` → copy) is usually the **`legacy_alter_table` PRAGMA default**, which is a compile-time setting of the *Python build*, NOT the sqlite version. When OFF (0) — standalone/Linux/CI builds — renaming a table rewrites foreign-key references to it in *other* tables' schemas (arming FK-repair migration paths); when ON (1) — e.g. macOS system Python 3.9 — it does not. Probe with `PRAGMA legacy_alter_table` + a rename-and-inspect. To get a real other-version env in seconds: `PYTHONPATH=<repo> uv run --no-project --python 3.12 <script>` (deps are stdlib for `database.py`); full pytest: `uv venv --python 3.12` + `uv pip install -r requirements.txt pytest pytest-asyncio==0.23.8 pytest-timeout`. Also: rebuild the FULL canonical schema and copy by **explicit shared-column name** (not `SELECT *`) — canonical column order differs from ADD COLUMN order, so `SELECT *` misaligns values even when counts match. | Session 39 (issue #3, CI migration bug) | Any migration/test that reproduces only on CI, or any sqlite table-rebuild in `database.py:init_db`. |
| 4 | Intermittent `sqlite3.OperationalError: disk I/O error` in the test suite (seen as an *error at fixture setup*, full-suite only, passes in isolation) = **stale WAL sidecars**. Connections run in WAL mode (`get_connection`, `database.py:22`), which creates `<db>-wal`/`<db>-shm`; SQLite only auto-removes them when the LAST connection closes. `reset_db` (`database.py:reset_db`) `os.remove`d only the main file, so when a concurrent connection was still open the stale `-wal`/`-shm` survived and mismatched the freshly recreated db → disk I/O error on `init_db`'s first read. Fix: delete `-wal`/`-shm`/`-journal` sidecars too. Repro is timing-dependent (needs a lingering connection at reset time), so verify by forcing a concurrent open connection, not by hoping the full suite flakes. `reset_db` is test-only (prod-safe). | Session 39 (test_admin_matches_page flake) | Any flaky "disk I/O error" / "database disk image is malformed" in tests, or any code that deletes/replaces a WAL sqlite file. |
