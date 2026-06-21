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
