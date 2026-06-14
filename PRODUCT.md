# WorkPilot — Product Document

> A personal AI work engine for a developer juggling multiple repos, Jira stories,
> and bugs. One place to start work, capture what happened, and recall it the next morning.

## The problem

A developer working across 10+ Git repos faces:
- Constant context switching — each repo needs a different mental model
- No memory across sessions — what you figured out today is gone tomorrow
- End-of-day fog — hard to remember what you actually did and decided
- Morning cold-start — you re-read old notes or start from scratch

Existing agents help with individual tasks but leave no trail. There is no "yesterday I
learned X in the payments repo" signal to surface the next morning.

## Who it's for

A software developer (the owner) who:
- Works across 10+ Git repos daily (mix of Java/Spring, Python, React, etc.)
- Juggles Jira stories, production bugs, and ad-hoc requests
- Uses the `claude` CLI on a local machine (no API key — subscription only)
- Wants one tool to start, track, and recall daily work

## The core loop

```
Morning  →  open Aide in browser → see yesterday's brief → know where you left off
Work     →  ./work.sh "PROJ-123 fix login timeout" payments-api → agent runs in repo
Evening  →  open Aide → add context to today's sessions → Claude writes the brief
```

Everything lives in `~/.workpilot/`. Nothing leaves the machine. No cloud sync, no external APIs.

## Principles

- **One command to start.** `./work.sh <task> <repo>` is all that's needed to kick off an
  agent session. No config, no project setup per repo.
- **Capture automatically, reflect manually.** The agent session is logged automatically
  (files changed, duration, outcome). The *meaning* — what was hard, what was decided —
  is added in the evening reflect. Both matter.
- **Morning recall is the payoff.** The product succeeds if, every morning, you open it
  and feel genuinely prepared — not just informed.
- **No API key, no cloud.** All LLM calls go through the `claude` CLI subprocess. The
  data store is plain files you own. Zero pip dependencies.
- **Repo registry, not repo coupling.** Aide knows your repos by nickname. It does not
  own or manage them — it works *inside* them as a guest.

## Architecture

```
workpilot/
  work.sh          CLI entry point: ./work.sh "<task>" <repo-nickname>
  server.py        stdlib HTTP server — morning brief, diary, reflect, repo registry
  ui/index.html    single-file web UI (no framework)
  app/
    config.py      paths (~/.workpilot/), CLAUDE_PATH, PORT
    llm.py         get_completion() via claude CLI subprocess (NO API KEY)
    agent.py       run_agent() — launches claude with tools inside a repo path
    store.py       file store: sessions, diary, briefs, repos registry
```

### CLI flow (`work.sh`)

```
./work.sh "PROJ-123: fix login timeout" payments-api
  1. Look up "payments-api" in ~/.workpilot/repos.json → get the path
  2. Run claude agent in that path with the task as the prompt
  3. Stream agent output to terminal (user watches it work)
  4. On completion: auto-write a session entry to ~/.workpilot/diary.jsonl
  5. Print a one-line summary: "Session done — 3 files changed in 8m 42s"
```

### Web server (port 7802)

```
GET  /                    → the UI
GET  /api/brief           → yesterday's brief (or empty state if none)
GET  /api/diary?date=     → all sessions for a date (default: today)
POST /api/reflect         → {date, sessions_with_notes} → Claude writes brief → saved
GET  /api/repos           → list registered repos
POST /api/repos           → {nickname, path, description} → register a repo
DELETE /api/repos/:name   → remove a repo
GET  /api/meta            → {model, workpilot_version}
```

### Store layout (`~/.workpilot/`)

```
repos.json              {nickname: {path, description, language}}
diary.jsonl             one JSON line per session (auto-written by work.sh)
briefs/YYYY-MM-DD.md    the evening synthesis for that day
sessions/<id>.json      full session record (task, files changed, duration, raw outcome)
```

#### Session record schema

```json
{
  "id": "sess-20240614-143022",
  "repo": "payments-api",
  "repo_path": "/home/user/work/payments-api",
  "task": "PROJ-123: fix login timeout",
  "started_at": "2024-06-14T14:30:22",
  "ended_at": "2024-06-14T14:38:54",
  "duration_s": 512,
  "files_changed": ["src/auth/LoginService.java", "src/auth/SessionConfig.java"],
  "num_turns": 7,
  "cost_usd": 0.41,
  "outcome": "<agent's final summary>",
  "notes": ""
}
```

`notes` is empty at write time; the evening reflect UI lets the user add context before
Claude synthesises the brief.

#### Diary entry (diary.jsonl)

One line per session — a compact subset of the session record for fast aggregation:

```json
{"id": "sess-...", "date": "2024-06-14", "repo": "payments-api",
 "task": "PROJ-123: fix login timeout", "duration_s": 512,
 "files_changed": 2, "cost_usd": 0.41}
```

#### Daily brief (`briefs/YYYY-MM-DD.md`)

Written by Claude from the session outcomes + user notes. Structure:

```markdown
# Brief — 14 June 2024

## What I worked on
- **payments-api** — fixed login timeout (PROJ-123). Root cause: session TTL was not
  being reset on activity. Changed SessionConfig + LoginService. Key learning: Spring
  Security's session registry needs explicit config for concurrent sessions.
- **user-service** — investigated PROJ-124 (not resolved). The issue is in the OAuth
  token refresh path; needs a fresh look tomorrow.

## Key decisions made
- Chose to extend session TTL at the service level, not the DB level — avoids a migration.

## Open / tomorrow
- PROJ-124 token refresh bug — pick up from OAuth2TokenRefreshFilter.
- Code review for payments PR #87 requested by team.

## Time
- Total: 3h 12m across 4 sessions
- Repos touched: payments-api, user-service
```

## Feature list

### v1 — Ship first

- [ ] **Repo registry** — `POST /api/repos` to register a repo by nickname + path. List + delete.
- [ ] **`work.sh`** — one command to run a claude agent session in a registered repo.
  Streams output. Writes diary entry + session file on completion.
- [ ] **Diary view** — web UI shows today's sessions: repo, task, duration, files changed.
  User can add notes to each session.
- [ ] **Evening reflect** — "Synthesise" button → Claude reads all sessions + notes →
  writes `briefs/YYYY-MM-DD.md`. Streamed or polled; server-side.
- [ ] **Morning brief** — landing page of the web UI shows yesterday's brief in full.
  If no brief exists, shows an empty state with a prompt to run `./work.sh`.
- [ ] **`start_up.sh` / `stop.sh`** — same pattern as review-dojo.

### v2 — Later

- Jira integration — read assigned tickets; link a session to a ticket ID; auto-update
  status to "In Progress" / "In Review" on session start/end. Needs Jira API token
  (store in `~/.workpilot/secrets.env`, never committed).
- Weekly retrospective — every Sunday, Claude reads the week's briefs and writes a
  `briefs/week-YYYY-WNN.md` with trends, repeated themes, growth edges.
- Multi-repo search — "where is the payment timeout config defined?" — Claude searches
  across all registered repos and returns file locations + relevant snippets.
- Calendar integration — pull today's meetings (Google Calendar) and surface them in
  the morning brief alongside yesterday's work.
- Voice reflect — speak your end-of-day notes via browser `SpeechRecognition` instead
  of typing (Phase 2 of what review-dojo's listen drill proved out).

## Agent runner (`app/agent.py`)

The core of the CLI path. Launches `claude` with tools inside the repo:

```python
def run_agent(repo_path, task, model="sonnet",
              allowed_tools="Read Edit Write Bash(git:*,mvn:*,npm:*,python:*)",
              max_budget_usd=3.0, timeout=900) -> dict:
    cmd = [
        CLAUDE_PATH,
        "--model", model,
        "--output-format", "json",
        "--allowedTools", allowed_tools,
        "--permission-mode", "acceptEdits",
        "--max-budget-usd", str(max_budget_usd),
        "-p", task,
    ]
    # run in repo_path, stream stdout to terminal, capture for session record
    # return {success, outcome, files_changed, num_turns, cost_usd, duration_s}
```

**Hard rules for the agent runner:**
- Always `acceptEdits`, never `bypassPermissions`
- Bash scope must be limited to build tools — never bare `Bash`
- `max_budget_usd` default is $3.00; `timeout` default is 900s (15 min)
- The CLI streams output live — the user watches the agent work, can Ctrl+C to stop
- On timeout or failure, still write the session file (partial record is useful)

## `work.sh` interface

```bash
./work.sh "<task description>" <repo-nickname>

# Examples:
./work.sh "PROJ-123: fix login session timeout" payments-api
./work.sh "review PR #87 and leave comments" payments-api
./work.sh "understand the OAuth refresh flow — read-only exploration" user-service

# Flags (optional, v2):
--budget 5.0       # override max spend for this session
--model haiku      # cheaper model for quick tasks
--dry-run          # print the command without running it
```

If the repo nickname is not in `repos.json`, print a helpful error:

```
✗ Repo "payments-api" not registered.
  Register it: open http://localhost:7802 → Repos tab → Add Repo
  Or: curl -X POST http://localhost:7802/api/repos \
        -d '{"nickname":"payments-api","path":"/path/to/repo","description":"..."}'
```

## Web UI structure

```
Aide
├── Brief      ← landing page: yesterday's brief (or empty state)
├── Today      ← diary for today: sessions list, add notes, Synthesise button
├── Repos      ← register / list / remove repos
└── History    ← past briefs by date (click to read)
```

Single-file HTML, same pattern as review-dojo's `ui/index.html`. No framework.
Hand-rolled markdown renderer for the brief display. Dark theme.

## How to run

```bash
# First time: register your repos
curl -X POST http://localhost:7802/api/repos \
  -H "Content-Type: application/json" \
  -d '{"nickname":"payments-api","path":"/absolute/path/to/payments-api","description":"Payment microservice"}'

# Every day
./start_up.sh          # starts the web server + opens browser
./work.sh "task" repo  # start an agent session (separate terminal)
# evening: open browser → Today tab → add notes → Synthesise
# Ctrl+C to stop server, or ./stop.sh
```

## Conventions

- Same terse, comment-the-why style as the rest of the codebase.
- Zero pip dependencies — Python stdlib only. The zero-dep property is a feature.
- No `ANTHROPIC_API_KEY` ever. All LLM calls via `claude` CLI subprocess.
- Port 7802 (avoids clash with review-dojo on 7800).
- All paths relative to `~/.workpilot/` — never hardcode absolute paths.
- `work.sh` is the CLI entry; `server.py` is the web entry. They share `app/` but are
  independent — the server does not need to be running for `work.sh` to work.

## Open questions

- Should `work.sh` auto-open the browser to the Today tab after a session ends?
- How to handle sessions that run longer than 15 min (raise budget? new session?)?
- Should the evening synthesise be triggered automatically (e.g. at 6 PM via cron)
  or always manual? Manual is safer — you want to add notes first.
- Jira: API token needed. Should it live in `~/.workpilot/secrets.env` (excluded from git)
  or in the OS keychain? OS keychain is safer but more setup.
