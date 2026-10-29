# Global agent guidance

Applies to every session in every project. Project-level `AGENTS.md` files add more specific rules
and take precedence where they conflict.

## MCP tool stack: how to call and when to use what

All MCP servers are reachable through the Code Mode `execute` runtime. Call tools with exact paths
from the catalog — `tools.<server>["tool-name"](input)` — never by guessed names. Inside `execute`,
use `search({ query, namespace?, limit? })` to discover tools and verify signatures; do not infer
signatures. The only listed external endpoints are previewed in the catalog; anything else goes
through `search(...)` first.

Routing decision table:

| Task                                          | Reach for                                  |
| --------------------------------------------- | ------------------------------------------ |
| Understand, navigate, or edit code of a project | serena                                    |
| Library/framework/SDK/API/CLI docs, migrations  | context7 (websearch only if context7 fails) |
| Ambiguity, hard debugging, multi-step planning  | sequential-thinking                        |
| Workouts, routines, body weight, PRs, balance   | opengym                                    |
| Worktrees, session location/rename              | built-in `tools.opencode.*`                |

Cross-cutting rules:

- Prefer the cheapest sufficient tool; never stack overlapping tools on the same target
  (e.g. native grep + serena `search_for_pattern` for one lookup).
- Batch independent tool calls in one turn; chain only genuinely dependent calls.
- Nothing here writes to opengym, and memory edits in serena need explicit user intent.

## serena — semantic code toolkit

Preconditions:

- Call `initial_instructions` before the first coding task; it provides the `session_id`
  that `activate_project` and subsequent calls require.
- Serena does **not** infer the active project from the current working directory. Before
  any project-scoped serena tool (`onboarding`, `list_memories`, symbol/file/edit tools,
  etc.), activate a project with `activate_project({ project, session })`, using the
  `session` from `initial_instructions`. If no project is active, those tools fail with
  "No active project".
- Required order is strict and one-way:
  1. `initial_instructions` → obtain `session_id`.
  2. `activate_project({ project, session })` → select one of the known projects, or ask
     the user for a project path if none fits.
  3. Only then call `onboarding`, `list_memories`, or any exploration/editing tool.
- `onboarding` is only valid after step 2; calling it earlier is the cause of the recurring
  "No active project" error.
- Locate memory briefing first: `list_memories` at the start of non-trivial project work;
  read only memories whose names look relevant.

Exploration (symbol-first, avoid reading whole files):

- File overview: `get_symbols_overview` before reading a new file.
- Named symbols: `find_symbol` (start with `depth=1`, `include_body=false`; add `include_body=true`
  only for symbols you are about to edit), `find_implementations`, `find_declaration`.
- References: `find_referencing_symbols` is the authoritative relationship check.
- Free-text search: `search_for_pattern` (regex + globs; use only when no named symbol fits).
- Files and layout: `find_file`, `list_dir`.
- Outside the active serena project, plain native `grep`/`glob`/`read` are preferable.

Editing hierarchy (pick the narrowest that fits):

- Small in-file edit: `replace_content` (regex mode with tight wildcards is cheapest).
- Whole-symbol rewrite: `replace_symbol_body` — only after a prior `include_body=true` read.
- Add code: `insert_after_symbol` / `insert_before_symbol` around the neighboring top-level symbol.
- Same edit across many files: `replace_in_files` — always `dry_run=true` first, then apply with
  `occurrence_ids` or a strict `expected_count`.
- Refactor: `rename_symbol`, `safe_delete_symbol` — trust the success result, do not re-verify
  by re-reading files or re-running builds just for the refactor itself.
- After meaningful code edits, check `get_diagnostics_for_file` when LSP diagnostics exist.

Gotchas:

- Every line number serena returns is 0-based.
- `execute_shell_command`: never launch long-running or interactive processes there; use the
  native shell tool for those.
- Memory writes/deletes/renames only when the user asked, or when project knowledge is durable
  and clearly useful (`write_memory`). Never delete memories unbidden.

## context7 — up-to-date library documentation

- Use for any question about a library, framework, SDK, API, CLI, or cloud service — even familiar
  ones; training data goes stale. This includes API syntax, config, version migration, setup, and
  debugging of library-specific behavior.
- Flow: `resolve-library-id` with the properly punctuated product name ("Next.js", not "nextjs";
  "Customer.io", not "customerio") → `query-docs` with the resolved ID.
- One concept per `query-docs` call; split multi-topic questions. Make queries specific
  ("React useEffect cleanup function", not "hooks"). Pass the user's programming language.
- Skip context7 for: business-logic debugging, code review, refactoring, writing code from
  scratch, and general programming concepts.
- Never include secrets, credentials, or proprietary code in queries.

## sequential-thinking — structured step-by-step reasoning

- Single tool: `tools["sequential-thinking"].sequentialthinking` with `thought`,
  `thoughtNumber`, `totalThoughts`, `nextThoughtNeeded`.
- Use for: ambiguous or underspecified requirements, bugs that resist the first hypothesis,
  ordering a multi-step plan, or a decision with several conflicting options.
- Keep each thought short and concrete; when correcting course, use `isRevision`/`revisesThought`
  instead of piling on new branches. `branchFromThought` only for real alternatives worth keeping.
- Stop as soon as a plan or conclusion is formed (`nextThoughtNeeded=false`).
- Never use it for single lookups, trivial edits, well-trodden workflows, or after planning
  is already done — it costs tokens and slows simple tasks.

## opengym — user's training log (read-only)

- Every opengym tool reads the user's fitness data; none writes it. Never promise to log,
  edit, or delete workouts — the app does that.
- Navigation pattern: `list_routines` → `get_routine` (full exercises and targets);
  `get_week_plan` is the authoritative answer to "what is planned when" (next 7 dates);
  `list_workouts` → `get_workout` (prefer `workout_id` when two sessions share a date);
  `preview_session` shows progression-adjusted numbers before the session starts.
- Analysis: `muscle_balance` (`week` | `month` | `all`) ranks muscles by effective sets;
  `estimate_1rm` uses Epley by default; `get_bodyweight` tracks weigh-ins vs goal.
- Dates are YYYY-MM-DD, inclusive ranges; the current date comes from context.

## OpenCode built-ins (brief)

- `worktree_create` / `worktree_list` / `worktree_remove`: parallel or throwaway work in isolated
  checkouts. Set `force` on removal only after the user confirmed discarding local changes.
- `session_move`: relocate a session's working directory (omit `sessionID` for the current one).
- `session_rename`: keep titles short and specific, summarizing the work.
- `list_mcp_resources` / `read_mcp_resource`: browse and read documents exposed by MCP servers.
- `models`: look up available models; set a model only when the user explicitly names one.
