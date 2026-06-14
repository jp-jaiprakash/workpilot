# WorkPilot — Implementation Plan

## Build Sequence

Foundation first (E01): config + store + llm must exist before anything else can run.
Agent CLI second (E02): `work.sh` + `agent.py` deliver the core value immediately —
the user can start logging sessions even before the web UI exists. Web backend (E03)
and UI (E04) come last and build on the data the agent already writes. No external
dependencies: zero pip installs, no API key, Python stdlib throughout.

## EPICS

```json
[
  {"id":"E01","title":"Core Foundation","goal":"Config, file store, and LLM subprocess wrapper — the plumbing everything else builds on","status":"active"},
  {"id":"E02","title":"Agent CLI","goal":"work.sh + agent.py so the user can run a Claude agent in any registered repo and get a diary entry written automatically","status":"active"},
  {"id":"E03","title":"Web Backend","goal":"stdlib HTTP server with all API endpoints: repos, diary, reflect, brief","status":"active"},
  {"id":"E04","title":"Web UI","goal":"Single-file dark-theme UI with Brief, Today, Repos, and History tabs","status":"active"},
  {"id":"E05","title":"Jira Integration","goal":"Read assigned tickets and link sessions to ticket IDs; auto-update status on session start/end","status":"future"},
  {"id":"E06","title":"Calendar Integration","goal":"Pull today's meetings from Google Calendar and surface them in the morning brief","status":"future"},
  {"id":"E07","title":"Weekly Retrospective","goal":"Sunday synthesis: read the week's briefs and write a weekly trends + growth-edges doc","status":"future"},
  {"id":"E08","title":"Multi-repo Search","goal":"Natural-language search across all registered repos — find where something is defined","status":"future"}
]
```

## STORIES

```json
[
  {
    "id": "S001",
    "epic_id": "E01",
    "title": "app/config.py — paths and constants",
    "user_story": "As the app, I need a single place for all paths and constants so nothing is hardcoded elsewhere",
    "acceptance_criteria": [
      "WORKPILOT_DIR defaults to ~/.workpilot/, overridable via WORKPILOT_DIR env var",
      "Subdirs REPOS_FILE, DIARY_FILE, SESSIONS_DIR, BRIEFS_DIR, QUEUE_DIR derived from WORKPILOT_DIR",
      "CLAUDE_PATH defaults to ~/.local/bin/claude, overridable via CLAUDE_PATH env var",
      "PORT=7802 overridable via WORKPILOT_PORT env var",
      "All directories created on import via os.makedirs(exist_ok=True)"
    ],
    "priority": "critical",
    "effort": "XS",
    "tags": ["backend"],
    "status": "todo"
  },
  {
    "id": "S002",
    "epic_id": "E01",
    "title": "app/store.py — file store for repos, diary, sessions, briefs",
    "user_story": "As the app, I need to read and write repos, session records, diary entries, and briefs from plain files so all data is owned by the user",
    "acceptance_criteria": [
      "repos_load() returns dict from repos.json; repos_save() writes it atomically",
      "repos_add(nickname, path, description, language) adds entry; repos_remove(nickname) removes it",
      "session_write(record: dict) writes sessions/<id>.json; session_load(id) reads it",
      "diary_append(entry: dict) appends one JSON line to diary.jsonl",
      "diary_for_date(date_str) returns list of session records for that date, enriched from sessions/ files",
      "brief_write(date_str, markdown) writes briefs/YYYY-MM-DD.md",
      "brief_read(date_str) returns markdown string or None if not found",
      "brief_list() returns list of date strings with existing briefs, newest first"
    ],
    "priority": "critical",
    "effort": "S",
    "tags": ["backend"],
    "status": "todo"
  },
  {
    "id": "S003",
    "epic_id": "E01",
    "title": "app/llm.py — get_completion() via claude CLI subprocess",
    "user_story": "As the app, I need a reliable wrapper around the claude CLI so all LLM calls work without an API key",
    "acceptance_criteria": [
      "get_completion(prompt, system, model, timeout) calls claude --output-format json -p in cwd=/tmp",
      "Parses JSON response and returns result string on success",
      "Raises QuotaExhaustedError with a clear message on rate limit / quota signals",
      "Raises RuntimeError with stderr + stdout on any other failure",
      "model defaults to sonnet; caller can override"
    ],
    "priority": "critical",
    "effort": "S",
    "tags": ["backend"],
    "status": "todo"
  },
  {
    "id": "S004",
    "epic_id": "E02",
    "title": "app/agent.py — run_agent() inside a repo with tools",
    "user_story": "As a developer, I need the agent to run inside my repo with real tools so it can read, edit, and build code autonomously",
    "acceptance_criteria": [
      "run_agent(repo_path, task, model, allowed_tools, max_budget_usd, timeout) launches claude with --permission-mode acceptEdits, never bypassPermissions",
      "allowed_tools default scopes Bash to build commands only: Bash(git:*,mvn:*,npm:*,python:*,gradle:*)",
      "max_budget_usd default 3.0, timeout default 900 — callers can raise these explicitly",
      "Captures git status --porcelain before and after to compute files_changed list",
      "Returns dict: {success, outcome, files_changed, num_turns, cost_usd, duration_s}",
      "On timeout (subprocess.TimeoutExpired) or non-JSON stdout, returns {success: False, ...} — never raises",
      "Calls store.session_write() and store.diary_append() on completion (even on failure)"
    ],
    "priority": "critical",
    "effort": "M",
    "tags": ["backend", "agent"],
    "status": "todo"
  },
  {
    "id": "S005",
    "epic_id": "E02",
    "title": "work.sh — CLI entry point to start an agent session",
    "user_story": "As a developer, I want one command to start an agent session in any repo so I don't have to think about paths or setup",
    "acceptance_criteria": [
      "./work.sh '<task>' <repo-nickname> starts an agent session and streams output to terminal",
      "Resolves nickname from ~/.workpilot/repos.json via python3 -c — no jq dependency",
      "Prints a clear error and exits non-zero if nickname is not registered, with the curl command to register it",
      "Prints one-line summary on completion: repo, files changed, duration, cost",
      "work.sh is executable (chmod +x) and has a #!/usr/bin/env bash shebang",
      "Does not require the web server to be running"
    ],
    "priority": "critical",
    "effort": "S",
    "tags": ["cli"],
    "status": "todo"
  },
  {
    "id": "S006",
    "epic_id": "E03",
    "title": "server.py — stdlib HTTP server skeleton",
    "user_story": "As the web UI, I need a server to call so I can read and write data",
    "acceptance_criteria": [
      "ThreadingHTTPServer on PORT from config, no framework, no pip deps",
      "_send(code, body, ctype) helper handles JSON and HTML responses",
      "GET / serves ui/index.html",
      "404 returned for unknown paths",
      "500 with {error: ...} for unhandled exceptions; traceback printed to stderr",
      "start_up.sh starts server + opens browser; stop.sh kills it and frees the port"
    ],
    "priority": "high",
    "effort": "S",
    "tags": ["backend"],
    "status": "todo"
  },
  {
    "id": "S007",
    "epic_id": "E03",
    "title": "GET/POST/DELETE /api/repos — repo registry endpoints",
    "user_story": "As a developer, I need to register and manage my repos from the web UI so work.sh can resolve them",
    "acceptance_criteria": [
      "GET /api/repos returns list of {nickname, path, description, language} objects",
      "POST /api/repos {nickname, path, description, language} adds repo; 400 if nickname or path missing",
      "DELETE /api/repos/<nickname> removes repo; 404 if not found",
      "All writes go through store.repos_save()"
    ],
    "priority": "high",
    "effort": "S",
    "tags": ["backend"],
    "status": "todo"
  },
  {
    "id": "S008",
    "epic_id": "E03",
    "title": "GET /api/diary — today's sessions endpoint",
    "user_story": "As the Today tab, I need to fetch today's sessions so I can show them to the user for annotation",
    "acceptance_criteria": [
      "GET /api/diary returns sessions for today (YYYY-MM-DD) by default",
      "GET /api/diary?date=YYYY-MM-DD returns sessions for that date",
      "Each item includes: id, repo, task, started_at, duration_s, files_changed list, cost_usd, notes",
      "Returns empty list [] when no sessions exist for the date — not a 404"
    ],
    "priority": "high",
    "effort": "S",
    "tags": ["backend"],
    "status": "todo"
  },
  {
    "id": "S009",
    "epic_id": "E03",
    "title": "POST /api/reflect — evening synthesis endpoint",
    "user_story": "As a developer, I want to hit Synthesise and have Claude write my daily brief so I don't have to do it manually",
    "acceptance_criteria": [
      "POST /api/reflect {date, sessions: [{id, notes}]} updates notes on session files then calls Claude",
      "Synthesis prompt asks Claude (sonnet) to write ## What I worked on / ## Key decisions made / ## Open & tomorrow / ## Time summary using actual filenames and ticket IDs",
      "Saves result to briefs/YYYY-MM-DD.md via store.brief_write()",
      "Returns {date, markdown} on success; {error: ...} on LLM failure",
      "QuotaExhaustedError surfaces as 429 with a clear message"
    ],
    "priority": "high",
    "effort": "M",
    "tags": ["backend"],
    "status": "todo"
  },
  {
    "id": "S010",
    "epic_id": "E03",
    "title": "GET /api/brief — morning brief endpoint",
    "user_story": "As the Brief tab, I need to fetch yesterday's brief so the user sees it first thing",
    "acceptance_criteria": [
      "GET /api/brief returns {date, markdown} for yesterday (today - 1 day)",
      "GET /api/brief?date=YYYY-MM-DD returns brief for that specific date",
      "Returns {date, markdown: null, empty: true} when no brief exists — not a 404",
      "GET /api/brief/list returns list of {date} objects with existing briefs, newest first"
    ],
    "priority": "high",
    "effort": "XS",
    "tags": ["backend"],
    "status": "todo"
  },
  {
    "id": "S011",
    "epic_id": "E04",
    "title": "ui/index.html — scaffold, dark theme, tabs, md() renderer",
    "user_story": "As a developer, I need a single-file web UI that opens in the browser so I don't need to install anything",
    "acceptance_criteria": [
      "Single HTML file, no framework, no CDN scripts — only Google Fonts",
      "Dark theme: #0d1117 background, #e6edf3 text, JetBrains Mono for code/mono elements",
      "Four tabs: Brief, Today, Repos, History — tab state managed in JS",
      "Hand-rolled md() function renders markdown headings, bold, bullets, horizontal rules to HTML",
      "Responsive layout, readable on a 13-inch laptop screen"
    ],
    "priority": "high",
    "effort": "S",
    "tags": ["frontend"],
    "status": "todo"
  },
  {
    "id": "S012",
    "epic_id": "E04",
    "title": "Brief tab — render yesterday's brief",
    "user_story": "As a developer, I want to see yesterday's brief the moment I open the app so I know where I left off",
    "acceptance_criteria": [
      "Brief tab is the default landing tab on app open",
      "Fetches GET /api/brief on load and renders markdown via md()",
      "Empty state when no brief: message 'No brief yet. Run ./work.sh to start a session, then Synthesise in the Today tab this evening.'",
      "Date of the brief shown as a subtitle",
      "Loading state shown while fetching"
    ],
    "priority": "high",
    "effort": "S",
    "tags": ["frontend"],
    "status": "todo"
  },
  {
    "id": "S013",
    "epic_id": "E04",
    "title": "Today tab — sessions list, notes, Synthesise button",
    "user_story": "As a developer, I want to annotate today's sessions and synthesise them into a brief so tomorrow-me knows what happened",
    "acceptance_criteria": [
      "Fetches GET /api/diary on tab open, shows each session as a card: repo badge, task text, duration in minutes, files changed count, cost",
      "Each session card has a notes textarea — empty by default, user types context",
      "Synthesise Day button at bottom calls POST /api/reflect with all session IDs and their current notes",
      "While synthesising: button shows 'Writing brief…' and is disabled",
      "On success: switches to Brief tab and renders the new brief",
      "Empty state when no sessions today: 'No sessions yet. Run ./work.sh to start one.'"
    ],
    "priority": "high",
    "effort": "M",
    "tags": ["frontend"],
    "status": "todo"
  },
  {
    "id": "S014",
    "epic_id": "E04",
    "title": "Repos tab — register, list, and remove repos",
    "user_story": "As a developer, I want to manage my repo registry from the UI so work.sh can find them",
    "acceptance_criteria": [
      "Fetches GET /api/repos on tab open; shows table with columns: Nickname, Path, Description, Language, Actions",
      "Add Repo form below the table: nickname (required), path (required), description, language select",
      "Submit calls POST /api/repos; refreshes list on success; shows inline error on failure",
      "Delete button on each row calls DELETE /api/repos/<nickname> with a confirmation prompt",
      "Empty state when no repos: shows the curl command to register a repo from the terminal"
    ],
    "priority": "high",
    "effort": "S",
    "tags": ["frontend"],
    "status": "todo"
  },
  {
    "id": "S015",
    "epic_id": "E04",
    "title": "History tab — past briefs by date",
    "user_story": "As a developer, I want to browse past briefs so I can recall what I worked on any day",
    "acceptance_criteria": [
      "Fetches GET /api/brief/list on tab open; shows list of dates newest first",
      "Clicking a date fetches GET /api/brief?date=YYYY-MM-DD and renders it inline via md()",
      "Back button returns to the date list",
      "Empty state when no briefs: 'No briefs yet. Complete a session and synthesise to create your first one.'"
    ],
    "priority": "medium",
    "effort": "S",
    "tags": ["frontend"],
    "status": "todo"
  },
  {
    "id": "S016",
    "epic_id": "E04",
    "title": "start_up.sh + stop.sh — server lifecycle scripts",
    "user_story": "As a developer, I want one command to start the app and one to stop it so there is no friction",
    "acceptance_criteria": [
      "start_up.sh kills any existing server on the port, starts python3 server.py, opens browser after 1.5s",
      "stop.sh kills the server process and frees the port, prints confirmation",
      "Both scripts are executable and work on macOS and Linux",
      "start_up.sh prints the URL before handing off to the server"
    ],
    "priority": "medium",
    "effort": "XS",
    "tags": ["cli"],
    "status": "todo"
  }
]
```
