# CLAUDE.md — WorkPilot

Personal AI work engine for a developer across multiple Git repos. One command to start
an agent session in any repo. End-of-day synthesis. Morning brief.

## Run it

```bash
./start_up.sh            # starts server + opens browser (port 7802)
./stop.sh                # kills the server
./work.sh "task" repo    # start an agent session (separate terminal, no server needed)
```

Zero pip dependencies — **Python stdlib only**. Requires the `claude` CLI on PATH.

## Hard rules (do not break these)

1. **No `ANTHROPIC_API_KEY`, ever.** All LLM calls go through the `claude` CLI
   subprocess in `app/llm.py` and `app/agent.py`. Use `sonnet` for synthesis (quality
   matters); `haiku` is acceptable for fast cheap ops.
2. **Agent tool scope must be restricted.** The agent runner (`app/agent.py`) must use
   `--permission-mode acceptEdits` (never `bypassPermissions`) and `--allowedTools`
   must scope Bash to specific commands (e.g. `Bash(git:*,mvn:*,npm:*)`), never bare
   `Bash`. Default budget $3.00, default timeout 900s.
3. **Repo registry is the source of truth.** `work.sh` resolves repo nicknames from
   `~/.workpilot/repos.json`. Never hardcode absolute paths in source files.
4. **Session files are append-only.** `diary.jsonl` is append-only. Session JSON files
   in `sessions/` are written once on completion and never overwritten (add a new record
   if re-run). Partial records (timeout, failure) are still written — a partial record
   is more useful than no record.
5. **`work.sh` and `server.py` are independent.** The web server does not need to be
   running for `work.sh` to work, and vice versa. They share `app/` but have no
   runtime coupling.

## Architecture

```
work.sh          CLI: ./work.sh "<task>" <repo-nickname>
start_up.sh      starts server + opens browser
stop.sh          kills server, frees port
server.py        stdlib HTTP server — diary, brief, reflect, repos API
ui/index.html    single-file web UI (no framework, dark theme)
app/
  config.py      paths (~/.workpilot/), CLAUDE_PATH, PORT=7802
  llm.py         get_completion() via claude CLI (copied pattern from review-dojo)
  agent.py       run_agent() — launches claude with tools inside a repo path
  store.py       repos.json, diary.jsonl, sessions/, briefs/
```

## Request flow

- `./work.sh "task" repo-nickname` → resolves path from repos.json → `agent.run_agent()`
  → streams output to terminal → writes `sessions/<id>.json` + appends to `diary.jsonl`
- `GET /api/brief` → reads `briefs/<yesterday>.md` → shown as morning landing page
- `GET /api/diary?date=YYYY-MM-DD` → reads diary.jsonl for that date + enriches from
  sessions/ files → returns list of sessions with notes field
- `POST /api/reflect` `{date, sessions: [{id, notes}]}` → Claude synthesises → writes
  `briefs/<date>.md` → returns the markdown
- `GET/POST/DELETE /api/repos` → manage `~/.workpilot/repos.json`

## Agent runner (`app/agent.py`)

Core of the CLI. See PRODUCT.md for the exact interface and hard rules.
The pattern is: `claude --output-format json --allowedTools "..." --permission-mode
acceptEdits --max-budget-usd 3.0 -p "<task>"` run with `cwd=repo_path`.

Parse the JSON output for: `result`, `total_cost_usd`, `num_turns`. Track changed files
via `git status --porcelain` before and after (diff the sets).

## Store layout (`~/.workpilot/`)

```
repos.json              {nickname: {path, description, language}}
diary.jsonl             compact line per session (date, repo, task, duration, files_changed, cost)
sessions/<id>.json      full session record (see PRODUCT.md for schema)
briefs/YYYY-MM-DD.md    evening synthesis for that day
```

## Web UI (`ui/index.html`)

Single-file, no framework, same dark-theme pattern as review-dojo. Tabs:
- **Brief** (landing): yesterday's brief in full. If empty, prompt to `./work.sh`.
- **Today**: list of today's sessions. Each has a notes textarea. "Synthesise" button
  calls `POST /api/reflect`.
- **Repos**: table of registered repos. Form to add new ones (nickname + path +
  description). Delete button.
- **History**: list of past brief dates → click to read.

Use a hand-rolled `md()` markdown renderer for the brief display (same as review-dojo).

## `work.sh`

Bash script. Not Python. Must:
1. Read `~/.workpilot/repos.json` (use `python3 -c` for the JSON parse — no jq dep)
2. Resolve nickname → path; error clearly if not found
3. `cd` to path and run `python3 <workpilot-dir>/app/agent.py "<task>"` (or call it inline)
4. On completion, print one-line summary

Keep it short — the logic lives in `app/agent.py`, not in the shell script.

## Evening synthesis prompt

When `POST /api/reflect` is called, send Claude a prompt like:

```
You are writing a personal work brief for a software developer.
Here are today's work sessions:
<foreach session: task, repo, files changed, duration, user notes>
Write a brief in this structure:
## What I worked on
## Key decisions made  
## Open / tomorrow
## Time summary
Be concrete. Use the actual file names and task IDs. Keep it under 300 words.
```

Use `sonnet` for this call — quality matters.

## Conventions

- Match the terse, comment-the-why style from review-dojo.
- Port 7802 (avoids clash with review-dojo on 7800).
- `work.sh` exits 0 on success, non-zero on failure.
- Dates are always `YYYY-MM-DD` (ISO). Times are local wall clock via `datetime.now()`.
- Session IDs: `sess-YYYYMMDD-HHMMSS` (timestamp-based, no UUID needed).
- See PRODUCT.md for the full feature list, v1 scope, and open questions.
