# TokenCost — Developer Notes

## Pre-deploy checklist

Before every `git push origin main`, verify:

### 1. No personal/local data in tracked files

Files that are known to contain local paths at runtime — must stay in `.gitignore`:

| File | Why dangerous |
|------|--------------|
| `.version_cache.json` | Contains `update_cmd` with absolute path to local repo |
| `tracker.db` | Contains all API request history |
| `*.log` | Contains request details |
| `.smart_routing` | Local flag file |

Run before committing:
```bash
grep -rn "$HOME\|/Users/" --include="*.py" --include="*.sh" --include="*.json" --include="*.md" . | grep -v ".gitignore"
```

If any hits — do NOT commit those files.

### 2. Bump the version

The auto-updater (`/api/update-status`) compares `VERSION` file on GitHub vs local.
Every push that changes behaviour must bump the version.

```bash
# Bump patch version (1.0.1 → 1.0.2)
echo "1.0.$(( $(cat VERSION | cut -d. -f3) + 1 ))" > VERSION
```

Update `RELEASE.md` with a short changelog for the new version.

### 3. Verify .gitignore covers runtime artifacts

```bash
git status --short | grep -v "^?? "   # should be clean or only tracked changes
git ls-files | xargs grep -l "/Users/" 2>/dev/null  # should be empty
```

---

## Project structure

| File | Purpose |
|------|---------|
| `proxy.py` | FastAPI proxy — intercepts all Anthropic API calls, logs to DB |
| `db.py` | SQLite helpers — `save_request`, `get_stats`, `init_db` |
| `import_history.py` | Imports historical data from local Claude/Cline/Copilot logs |
| `optimizer.py` | Request optimizations (cache injection, smart routing) |
| `dashboard.html` | Single-file frontend — fetches `/stats`, `/projects` etc. |
| `onbording.sh` | Install/update script — sets up launchd daemons |
| `VERSION` | Plain text semver, read by updater |
| `RELEASE.md` | Human-readable changelog for latest version |

## Key endpoints

- `GET /stats?period=today|7d|30d|all` — main dashboard data
- `POST /sync-now` — trigger import_history.py manually
- `GET /api/update-status` — compare local vs GitHub VERSION
- `POST /api/update` — pull + restart via onbording.sh

## Sync / import pipeline

```
Claude CLI JSONL  ─┐
Claude Desktop    ─┤
OpenClaw          ─┼─► import_history.py ──► tracker.db ◄── proxy.py (live)
Cline/Roo         ─┤
Copilot logs      ─┘
```

- Proxy captures live requests (vscode, claude-cli routed through proxy)
- `import_history.py` fills historical data with dedup via `msg_uuid` UNIQUE index
- Cutoff prevents double-counting: history import skips records newer than first proxy capture
