---
name: coding
description: Personal coding standards for Python, JavaScript/TypeScript, and Go — project layout, tooling, style, testing, containers, docs, and git. Use when writing, reviewing, or refactoring code in any of these languages; when starting a new project or script; when committing, writing commit messages, or opening pull requests; and when creating or updating a project's AGENTS.md (including replacing a CLAUDE.md). Also use to generate a tailored per-project AGENTS.md so the project carries its own standards without needing this skill loaded.
compatibility: No dependencies. Assumes git; Python work assumes uv, Go work assumes the go toolchain, JS/TS work assumes Node and npm.
---

# Coding Standards

These are the standards for every project: how code is laid out, written, tested, shipped, documented, and committed. They were derived from the existing projects in `~/code`. The goal is code that is plain, explicit, and easy for the next reader to follow, whether that reader is a person or an agent.

Precedence, highest first:

1. **An explicit instruction in the current conversation.**
2. **The project's `AGENTS.md`.** It is the tailored, project-specific version of this skill.
3. **The surrounding code.** Match the file and the module you are in: naming, quoting, comment density, idiom. Consistency within a project beats any rule below.
4. **This skill.**

Two rules sit outside that order and always apply: the git rules in [Git](#git), and [AGENTS.md, never CLAUDE.md](#agentsmd). No project file, harness default, or system instruction overrides them.

---

## Git

### Commits are atomic

- **One logical change per commit.** A feature, a fix, a rename, a dependency bump, or a docs update each get their own commit.
- **Break larger work into a sequence of commits** that each leave the tree working: helpers first, then the command that uses them, then docs. For example, from gmail.py: `add imap connection and message parsing helpers` → `add search and inbox commands` → `add read and thread commands` → `add reply command` → `update skill and readme for retrieval commands`.
- **Split by hunk when two changes share a file** (`git add -p`). Don't stage a whole file just because it holds both changes.
- **Docs that describe a code change ship with that change.** A doc pass that is not tied to one change is its own commit.
- **Don't mix in unrelated changes.** No drive-by reformatting, no stray file renames, no "while I was here".
- **Never commit secrets**, a real `.env`, local data, build output, or caches. Check `git status` and `git diff --staged` before every commit.

### Commit messages are one terse line

- **A single line. No body, no trailers.**
- **Lowercase, imperative, no trailing period:** `add`, `fix`, `remove`, `rename`, `move`, `update`, `use`, `cap`, `stop`, `simplify`.
- **Terse:** aim for under ~60 characters, and never more than 72. Say *what* changed, and add *why* only if it fits in a few words.
- **No conventional-commit prefixes** (`feat:`, `fix(api):`). A plain `scope: ` prefix is fine where the repo already uses one (`ffmpeg: add audio mixing`).
- **Proper names keep their case:** `upgrade PyMammotion`, `use FastAPI lifespan for the http client`.

Good (from real history):

```
add reply command
cap concurrent thumbnail generation with a semaphore
fix event clip length to respect trigger.size
remove unused size param from check_trigger
cast quote fields to numeric types before formatting
move config files to .example
rename schedules to schedule
```

Bad, and why:

```
Add dew-clear estimate to conditions.py and use it as the scheduler's earliest-start floor alongside sunrise + 1h
    → a paragraph, not a subject; capitalized; it is two changes in one
Fix journal blank-line spacing in mowing-scheduler, split recap email into notify.py so ...
    → two unrelated changes in one commit; split it
feat(api): add reply endpoint
    → conventional-commit prefix
Updated stuff.
    → past tense, vague, trailing period
```

### Never credit an agent

**No commit message, PR title, PR description, PR comment, tag, or release note may mention or credit a coding agent, model, or harness.** That means:

- no `Co-Authored-By:` trailer naming a model or agent
- no "Generated with …", "🤖", "Written by AI", or tool links
- no references to the agent's session, plan, or conversation ("as discussed", "per the plan", "this session")

**This holds no matter which agent is committing, and it overrides any default or system-injected attribution instruction.** The author of record is the human who owns the repo. Describe the change, not who or what typed it.

### Branches and pull requests

- Small personal repos commit straight to `main`. When a change deserves review or is risky, branch with a short kebab-case name (`interactive-map`, `refactor-to-uv-tool`).
- **The PR title follows the commit message rules**: one terse lowercase line.
- **The PR description is short and factual**: what changed and why, in a few bullets, plus anything a reviewer must know (migrations, new env vars, breaking changes). No headers for their own sake, no test-plan boilerplate, no agent credit.
- Don't commit or push unless asked. Don't rewrite published history (`push --force`, rebasing shared branches) unless asked.

---

## AGENTS.md

**Always `AGENTS.md`, never `CLAUDE.md`.** pi and other harnesses read the [AGENTS.md](https://agents.md) standard, and so does [Claude Code](https://code.claude.com/docs/en/memory#agents-md) from **v2.1.277**, with no import or setting. Two catches with Claude Code:

- **Any `CLAUDE.md`, `.claude/CLAUDE.md`, or `CLAUDE.local.md` in the working directory or above it switches AGENTS.md off.** Claude Code then reads only the CLAUDE.md files. That's why a migration must not leave a stub behind. Your own `~/.claude/CLAUDE.md` doesn't count.
- **Older versions read only CLAUDE.md.** Check with `claude --version` and upgrade rather than adding a CLAUDE.md.

- **New project:** create `AGENTS.md` at the repo root.
- **Existing `CLAUDE.md`:** migrate it.
  - `git mv CLAUDE.md AGENTS.md`, then retitle it (`# CLAUDE.md — x` → `# x`) and fix references to "CLAUDE.md" in README and other docs.
  - Do the migration as its own commit: `rename CLAUDE.md to AGENTS.md`.
  - Don't leave a `CLAUDE.md` symlink or stub behind.
- **Subdirectories** may have their own `AGENTS.md` when one part of the tree has distinct rules (e.g. `pi/AGENTS.md` for an agent runtime's instructions). The nearest file wins.
- **When docs say "AGENTS.md", update it in the same change as the code it describes.** A stale `AGENTS.md` is worse than none.

### Generating a per-project AGENTS.md

A project's `AGENTS.md` is this skill **tailored to the project and made standalone**. It has to work in a session where this skill is never loaded. So:

1. **Survey the repo first.** Read the tracked files (`git ls-files`), the manifests (`pyproject.toml`, `package.json`, `go.mod`), the Dockerfile and compose file, CI workflows, the README, any existing `CLAUDE.md`/`AGENTS.md`, and `git log --format=%s -30`.
2. **Find the real commands.** Use the ones that actually work in this repo for running, testing, linting, formatting, type-checking, migrating, and building. Run them if you can.
3. **Include only the sections that apply.** A Go-only repo gets no Python section. A single-file uv script gets the CLI conventions, not the service ones.
4. **Copy the [Git](#git) rules in, condensed but complete.** That means atomic commits, the one-line lowercase imperative message, and never crediting an agent. This is non-negotiable, and it is the most important part of the file.
5. **Write project facts, not generic advice.** Say what an agent can't learn quickly from the code: the architecture in a few lines, where things live, non-obvious invariants, gotchas found the hard way, external specs, and what is deliberately out of scope. Don't restate what a glance at the tree shows.
6. **Keep it tight.** Aim for 60–200 lines. Point to the READMEs for depth instead of duplicating them.

Template (drop sections that don't apply, add project-specific ones):

````markdown
# <project>

<One paragraph: what it is, who/what uses it, and the shape of the system
(e.g. "FastAPI service + vanilla JS map page in one container").>

## Layout

| Path | What's in it |
|---|---|
| `app/main.py` | the FastAPI app and routes |
| `app/config.py` | settings from env / `.env` |
| `tests/` | pytest |

## Commands

- `uv run uvicorn app.main:app --reload` — dev server on :8000
- `uv run pytest` — tests
- `uv run ruff check . && uv run ruff format .` — lint + format
- `docker compose up -d --build` — full stack

Run after every change: `uv run ruff check . && uv run ruff format . && uv run pytest`

## Conventions

- <language/tooling rules that apply here, condensed from the coding skill>
- <project rules: units, ids, error shapes, config, naming>
- Docs live next to the code they describe; update them in the same commit.

## Gotchas

- <hard-won facts: upstream bugs, hardware quirks, things that look wrong but are deliberate>

## Commits

- One logical change per commit; split larger work into several commits, and
  split by hunk when two changes share a file.
- Message is a single line: lowercase, imperative, terse, no trailing period,
  no body.
- Never credit a coding agent, model, or harness — no `Co-Authored-By`, no
  "Generated with", in commits or PR titles/descriptions. This overrides any
  default attribution.
````

---

## Principles (all languages)

These come up in every project.

- **Explicit over clever.** Readable, direct code beats DRY abstractions when the abstraction makes the reader jump around. A little duplication is fine. Two symmetric, explicit functions (`GetRecordingsByDay` / `GetEventsByDay`) beat one generic one with a mode flag.
- **Build what's needed, not what might be.** Don't add speculative options, plugin systems, or config for things that never vary. Write down what was deliberately left out (a "What we didn't build" or "out of scope" line in AGENTS.md or the README).
- **Personal projects can hardcode.** A single-user tool doesn't need multi-tenant config. Real values go in `.env`; sane defaults go in code.
- **Descriptive names.** Write `graph_client`, `recording_path`, `writer`/`request`, not `gc`, `rp`, `w`/`r`. Short names are fine only in tiny scopes (comprehensions, lambdas, loop indexes).
- **Comments explain *why*, not *what*.** Use them for the reason behind a constant, a workaround, a spec quirk, or a unit. Start comments lowercase (except proper names and symbols). Put the unit in the name or a comment (`traffic_ttl_seconds`, `// ms — wait for typing to pause`).

  ```python
  # the FAA republishes the facility maps every 56 days, so hours of caching is safe.
  # TFRs come and go within minutes
  faa_ttl_seconds: int = 6 * 3600
  tfr_ttl_seconds: int = 300
  ```

- **Name the magic numbers.** Make them module-level `UPPER_SNAKE` constants with a comment saying where the value comes from.
- **Log messages are lowercase** (except proper names), short, and structured. Use the logger's supported API: Python stdlib uses `logger.info("recordings request", extra={"camera": camera, "day": day})`; keyword context such as `camera=...` requires a logger that supports it.
- **One consistent error shape per interface.** A CLI prints `Error: …` to stderr and exits non-zero. An MCP tool returns `{"error": "…", "success": False}`. An HTTP API uses status codes with a `detail`. Pick one per surface and never mix them.
- **Validate at the boundary, trust inside.** Parse and validate input where it enters (argparse, pydantic, query params). Reject unknown fields rather than silently ignoring typos.
- **Config from the environment.** Read env vars, then `.env`. Commit a documented template (`.env.template`, or `.env.example` where the repo already uses that) with every setting, its default, and a comment. Secrets live only in `.env`, which is always gitignored.
- **Delete, don't comment out.** Dead code, unused params, and commented-out blocks go. Git has the history.
- **Every file ends with a newline.** No trailing whitespace.
- **Docs live next to the code they describe** (see [Documentation](#documentation)), and change in the same commit as the code.

---

## Python

### Tooling

| Concern | Use |
|---|---|
| Python | 3.13+ for new projects (`requires-python = ">=3.13"`); single-file scripts may declare `>=3.10`/`>=3.12` when their deps allow |
| Env / deps / running | [uv](https://docs.astral.sh/uv/) — `uv run`, `uv sync`, `uv add`, committed `uv.lock`. Not pip, Pipenv, Poetry, or setup.py for new work |
| Lint + format | [ruff](https://docs.astral.sh/ruff/) — `uv run ruff check . && uv run ruff format .`, `line-length = 100` |
| Tests | [pytest](https://docs.pytest.org/) |
| Settings | `pydantic-settings` `BaseSettings` |
| HTTP client | `httpx` (async in services); `requests` is fine in simple sync scripts |
| Web API | [FastAPI](https://fastapi.tiangolo.com/) + uvicorn |
| DB | SQLAlchemy 2.0 + Alembic (+ GeoAlchemy2/PostGIS for geo) |
| MCP servers | FastMCP |
| CLI parsing | `argparse` with subcommands (stdlib, zero deps); `typer` + `rich` acceptable for richer interactive CLIs |

### Archetype 1 — single-file uv script

This is for agent-facing CLIs and small tools (`geo.py`, `gmail.py`, `stocks.py`, `mammotion.py`). One file, no package, with deps declared inline via [PEP 723](https://peps.python.org/pep-0723/). Ship it with a `SKILL.md` (for agents) and a `README.md` (for people).

Repo layout:

```
gmail.py/
├── gmail.py          # the whole CLI (or scripts/<name>.py when packaged as a skill zip)
├── SKILL.md          # agent-facing contract
├── README.md         # human-facing docs
├── AGENTS.md         # dev notes for agents working on the tool
├── tests/            # optional, test_<name>.py
└── .gitignore
```

File structure, top to bottom:

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.12"
# dependencies = [
#     "httpx>=0.28",
# ]
# ///
"""
tool.py - one line on what it does.

A sentence or two on who it's for (agents and humans) and what it covers.
"""

import argparse
import json
import os
import sys

import httpx

# why this value: where it comes from, what it trades off
DEFAULT_LIMIT = 20

CLI_EPILOG = """\
Examples:
  tool.py list --limit 10
  tool.py get 4821 --json

Setup:
  export TOOL_TOKEN="..."
"""


class ToolError(Exception):
    """An error worth showing the user as-is."""


class ToolClient:
    """All I/O against the service. Commands never talk to the network directly."""

    def __init__(self, token: str):
        self.http = httpx.Client(headers={"Authorization": f"Bearer {token}"}, timeout=30)

    def __enter__(self):
        return self

    def __exit__(self, *exc):
        self.http.close()

    def list_items(self, limit: int) -> list[dict]:
        ...


def cmd_list(client: ToolClient, args: argparse.Namespace) -> int:
    items = client.list_items(args.limit)
    if args.json:
        print(json.dumps(items, indent=2))
        return 0
    for item in items:
        print(f"{item['id']}  {item['name']}")
    return 0


def main() -> int:
    parser = argparse.ArgumentParser(
        description=__doc__.strip().splitlines()[0],
        epilog=CLI_EPILOG,
        formatter_class=argparse.RawDescriptionHelpFormatter,
    )
    subparsers = parser.add_subparsers(dest="command")

    list_parser = subparsers.add_parser("list", help="List items")
    list_parser.add_argument(
        "--limit", type=int, default=DEFAULT_LIMIT, help=f"Max items (default: {DEFAULT_LIMIT})"
    )
    list_parser.add_argument("--json", action="store_true", help="Output JSON")

    args = parser.parse_args()
    if not args.command:
        parser.print_help()
        return 1

    token = os.environ.get("TOOL_TOKEN")
    if not token:
        print("Error: TOOL_TOKEN is not set", file=sys.stderr)
        return 1

    commands = {"list": cmd_list}
    with ToolClient(token) as client:
        try:
            return commands[args.command](client, args)
        except ToolError as e:
            print(f"Error: {e}", file=sys.stderr)
            return 1


if __name__ == "__main__":
    sys.exit(main())
```

Conventions for these CLIs:

- **Design for agents first.** Use predictable subcommands, a `--json` flag on every read command, and stable output fields. Make output compact and parseable, with no spinners or color when not on a TTY. Put the exact fields an agent needs in `SKILL.md`.
- **`CLI_EPILOG` holds real, copy-pasteable examples** for every command, plus setup steps.
- **One `XxxError` exception** for user-facing failures. **One `XxxClient` class**, used as a context manager, owns all I/O. **`cmd_<name>(client, args) -> int`** handlers are dispatched from a dict. `main()` returns the exit code.
- **Errors go to stderr, prefixed `Error:`**, and exit 1. Data goes to stdout. Warnings (`Warning: …`) go to stderr.
- **Credentials come from env vars** (`TOOL_USER`, `TOOL_TOKEN`), optionally overridable by a flag. Say exactly which var is missing and how to get it.
- **Destructive or outbound actions get `--dry-run`.**
- **Module-name shadowing:** a script shadows an import only when its module name collides with it: `roborock.py` importing the `roborock` library imports itself. Prefer renaming a colliding file. When the CLI name must stay, strip the script's own directory from `sys.path` (and pop any half-imported module from `sys.modules`) before the import, with a comment saying why and "do not remove".
- **Add a command in one change:** client method → `cmd_` handler → subparser + dispatch entry → epilog example → `SKILL.md` → `README.md`.

### Archetype 2 — service / package

This is for anything long-running or multi-module: FastAPI services, MCP servers, workers (`kestrel-map`, `dji-planner/api`, `obsidian-mcp`).

```
kestrel-map/
├── app/                  # or src/<package>/ for an installable package/tool
│   ├── __init__.py
│   ├── main.py           # FastAPI app, lifespan, routes (or routes/ once it grows)
│   ├── config.py         # Settings(BaseSettings) + module-level `settings`
│   ├── cache.py          # one module per concern
│   └── static/           # index.html, app.js, app.css — no-build UI
├── tests/
│   └── test_<module>.py
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── compose.yaml
├── .env.template
├── .dockerignore
├── .gitignore
├── README.md
└── AGENTS.md
```

When the API grows, layer it as `routes/` (HTTP only), `schemas/` (pydantic request/response), `services/` (logic and persistence), and `models/` (SQLAlchemy tables). Routes call services. Services never import from routes.

`pyproject.toml`:

```toml
[project]
name = "kestrel-map"
version = "0.1.0"
description = "One line, the same as the README's first line."
requires-python = ">=3.13"
dependencies = [
    "fastapi>=0.141.1",
    "httpx>=0.28.1",
    "pydantic-settings>=2.15.0",
    "uvicorn[standard]>=0.52.4",
]

[dependency-groups]
dev = [
    "pytest>=9.1.1",
    "ruff>=0.16.7",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]

[tool.ruff]
line-length = 100
target-version = "py313"
```

For an installable tool (`uv tool install`), use a `src/<package>/` layout with hatchling and a `[project.scripts]` entry point.

Settings:

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="KESTREL_", env_file=".env", extra="ignore")

    # where the map opens the first time
    center: Annotated[tuple[float, float], NoDecode] = (41.786, -87.752)
    traffic_ttl_seconds: int = 2


settings = Settings()
```

Service conventions:

- **Shared clients are created in the FastAPI `lifespan`** and hung on `app.state` (one `httpx.AsyncClient` with a timeout and User-Agent, closed on shutdown). Don't create a client per request.
- **Wrap each upstream in its own client class** with its own error type (`FAAClient`/`FAAError`). Routes translate those errors to HTTP statuses.
- **Use `Annotated[...]` for dependencies and query constraints** (`DbSession = Annotated[Session, Depends(get_session)]`, `Query(ge=0, le=200)`). Give return-type hints to response schemas.
- **Strict input:** pydantic models reject unknown fields (`extra="forbid"`) for request bodies.
- **Every container service has a health endpoint** returning `{"ok": True}`. Use the name the project already has (`/healthz` in kestrel-map, `/health` in dji-planner and securitron); new projects use `/healthz`.
- **Log with `logging.getLogger("<area>")`**, using lowercase messages. Don't `print` in services.
- **Name units:** `_seconds`, `_ms`, `_m`, `_ft`. Document the units in the API README.
- **Migrations:** `uv run alembic revision --autogenerate -m "..."`, then check `uv run alembic check` in the run-after-changes list. Import every model in `models/__init__.py` so autogenerate sees it.

### Python style

- **Formatting is whatever `ruff format` produces** (double quotes, `"""` docstrings) in new projects. Older projects that use single quotes and `'''` docstrings keep them. Match the file.
- **Type-hint every function signature.** Use modern syntax: `list[str]`, `dict[str, Any]`, `X | None` (not `Optional[X]`/`List[X]` in new code), and `collections.abc` for `Callable`/`Awaitable`.
- **Docstrings** go on modules, classes, and non-obvious public functions. The first line is a sentence about what it does, and further paragraphs say why and how. Skip docstrings that restate the name.
- **Private helpers are prefixed `_`.** Module constants are `UPPER_SNAKE`. Classes are `PascalCase`. Everything else is `snake_case`.
- **Use `@dataclass(frozen=True)` for internal records** and pydantic models at boundaries (API bodies, settings, MCP inputs). Use plain dicts only for JSON pass-through.
- **Use `pathlib.Path`, not `os.path`.** Use `datetime` with tz (`datetime.now(UTC)`). Never use naive local times in stored data.
- **Imports go in three groups** (stdlib, third-party, local), sorted, one per line for `from` imports when ruff's isort is on. No wildcard imports. Import modules, not names, when it reads better (`from app import geocode, traffic`).
- **Chain exceptions** (`raise X(...) from e`, or `from None` when the cause is noise). Never use a bare `except:`. Catch the narrowest type.
- **No `if __name__ == "__main__"` blocks in library modules.** Entry points live in `main()`/`run()`.

### Python testing

- **pytest, with tests in `tests/test_<module>.py`** and functions named `test_<behavior>` (`test_airspace_tile_is_styled_and_cached`).
- **Prefer hand-written fakes over mocks.** A small `FakeFAA` class with the same async methods, recording calls and toggling failure, beats `MagicMock` chains. Swap it onto `app.state` inside a fixture.
- **For FastAPI, use `TestClient(app)` as a context manager** so the lifespan runs.
- **For HTTP client libraries, record real responses with `pytest-vcr`** cassettes. Filter auth tokens out of the cassettes.
- **For filesystem code, use `tmp_path`** and build test data on the fly. Don't check in fixture trees unless they're real samples.
- **Tests that need a database or hardware skip cleanly** when it's unavailable. They never fail the default run.
- **Test behavior through the public surface** (HTTP endpoints, CLI commands, client methods), not private helpers.

---

## JavaScript / TypeScript

### Choosing the shape

- **No-build vanilla JS** is the default for small UIs served by a Python or Go backend (`app/static/index.html`, `app.js`, `app.css`). Load libraries (Leaflet etc.) from a pinned CDN URL or vendor them. No bundler, no `node_modules`.
- **TypeScript + React + Vite** is for real applications with state, routing, or many components (`dji-planner/ui`, `tapes/ui`). New apps are always TypeScript, never plain JS.

### Tooling (TS apps)

| Concern | Use |
|---|---|
| Build / dev server | [Vite](https://vite.dev/) — `npm run dev`, `npm run build` (`tsc -b && vite build`) |
| UI | React 19 function components + hooks |
| Types | [TypeScript](https://www.typescriptlang.org/) `strict` — `npm run typecheck` (`tsc -b`) |
| Lint | [oxlint](https://oxc.rs/docs/guide/usage/linter) — `npm run lint` |
| Packages | npm with a committed `package-lock.json`; `npm ci` in Docker/CI |
| HTTP | native `fetch` behind a small typed client — not axios in new code |

`package.json` scripts are always `dev`, `build`, `lint`, `typecheck`, `preview`. Keep dependencies minimal. Every dependency must earn its place.

tsconfig essentials: `"strict": true`, `"noUnusedLocals"`, `"noUnusedParameters"`, `"noFallthroughCasesInSwitch"`, `"verbatimModuleSyntax"`, `"moduleResolution": "bundler"`, `"allowImportingTsExtensions"` (imports carry `.ts`/`.tsx`), `"erasableSyntaxOnly"` (no enums or namespaces; use string-literal unions).

In dev, Vite proxies `/api` to the backend so the browser always uses same-origin paths. The UI reads `VITE_*` vars from the repo-root `.env` (`envDir: '..'`).

### Layout (TS apps)

```
ui/src/
├── main.tsx             # mount
├── App.tsx
├── api/
│   ├── client.ts        # request<T>() + one exported function per endpoint
│   └── types.ts         # wire types mirroring the API schemas
├── components/          # PascalCase.tsx, one component per file
├── editor/              # domain state + hooks (useMissionList.ts, draft.ts)
└── index.css
```

### TS style

- **2-space indent, single quotes, no semicolons, trailing commas** in multi-line literals. Keep line length near 100.
- **`type` over `interface`** for object shapes. Use **discriminated unions** with a literal `type`/`kind` field for variants (`type: 'recording' | 'event'`).
- **`import type { … }`** for type-only imports.
- **Pass self-contained objects, not prop lists:** `<VideoPlayer media={item} />`, not `<VideoPlayer camera={…} timestamp={…} type={…} />`.
- **One small typed fetch wrapper** with an `ApiError` class carrying `status` and `detail`, plus one function per endpoint taking typed params and an optional `AbortSignal`:

  ```ts
  export class ApiError extends Error {
    readonly status: number
    readonly detail: unknown

    constructor(status: number, detail: unknown) {
      super(typeof detail === 'string' ? detail : `request failed (${status})`)
      this.status = status
      this.detail = detail
    }
  }

  async function request<T>(path: string, init?: RequestInit): Promise<T> {
    const headers = new Headers(init?.headers)
    if (typeof init?.body === 'string' && !headers.has('content-type')) {
      headers.set('content-type', 'application/json')
    }
    const res = await fetch(`/api${path}`, { ...init, headers })
    if (res.status === 204) return undefined as T
    const body = await res.json().catch(() => null)
    if (!res.ok) throw new ApiError(res.status, body?.detail)
    return body as T
  }
  ```

- **Data-loading hooks** own their state (`{ items, total, loading, error }`). They cancel with `AbortController` in the effect cleanup, ignore `AbortError`, and debounce user input with a named delay constant.
- **Named exports** for everything except where a framework needs a default.
- **Comments are lowercase and say why:** `// a PUT refused because someone else saved since this copy was loaded`.

### Vanilla JS style

- Use modern ES (`const`/`let`, never `var`), arrow functions, template literals, `async`/`await` with `fetch`.
- **Start the file with a header comment** explaining what the page does and how data flows. Split sections with a `// ----` rule plus a short comment.
- **Constants at the top** in `UPPER_SNAKE` with units (`RADAR_REFRESH_MS`).
- **`localStorage` is a convenience that can throw.** Wrap every read and write in `try/catch` and fall back to defaults.
- **Escape anything interpolated into HTML** with a small `esc()` helper. Never put raw `innerHTML` on remote data.
- **Get config from the backend** (`/api/config`), not hardcoded in the JS.
- Match the file's existing semicolon style. Existing vanilla files use semicolons.

---

## Go

### Tooling

- **Go 1.22+**; new modules use the current stable release. `gofmt` (or `go fmt ./...`) and `go vet ./...` must be clean.
- **Module path `github.com/vicgarcia/<repo>`.**
- **Stdlib first:** `log/slog`, `embed`, `encoding/json`. Existing projects route with gorilla/mux. For a new module, prefer the stdlib router's method/pattern routing from Go 1.22 (`mux.HandleFunc("GET /cameras/{camera}/recordings", …)`, `request.PathValue("camera")`). That is a recommendation, not existing practice. Don't migrate working gorilla/mux code without a reason.
- **Few dependencies.** Each one needs a reason (gocv for OpenCV, a JWT library for auth).

### Layout

```
api/
├── main.go              # router, server, background workers, graceful shutdown
├── go.mod / go.sum
├── dist/                # built UI, embedded (gitignored, produced by the build)
└── internal/
    ├── auth/auth.go
    ├── cameras/cameras.go
    ├── env/env.go       # GetString / GetInt with defaults
    ├── logger/logger.go # slog setup + Info/Debug/Error helpers
    ├── handlers/        # one file per domain: health.go, auth.go, recordings.go, events.go
    └── media/           # domain logic + types.go
```

- **Everything under `internal/`**, one package per domain. Packages are short, lowercase, single words.
- **Handlers stay thin:** parse the request, call the domain package, write the response. **Domain functions return errors.** The handler decides the HTTP status.
- **Keep APIs symmetric across sibling resources:** `GetRecordingsByDay`/`GetEventsByDay`, `GetRecordingPath`/`GetEventPath`.
- **Ship a single binary.** Embed the built UI with `//go:embed dist` and serve it from `/`.

### Go style

- **Doc comment on every exported identifier,** starting with its name: `// GetRecordingsByDay returns recordings for a specific camera and day`.
- **Descriptive names,** including in handlers: `func RecordingsHandler(writer http.ResponseWriter, request *http.Request)`.
- **Always `return` after writing an error response.** A missing `return` after `http.Error` is a real bug in past code.
- **Handle each error once:** domain code returns it, and the handler logs it and writes the response. When an error crosses a package boundary, add context with `fmt.Errorf("reading %s: %w", path, err)`. That's recommended; existing code mostly returns errors bare.
- **Logging is structured `slog`** with lowercase messages and key/value pairs: `logger.Error("error querying recordings", "camera", name, "day", day, "error", err)`. Debug level is toggled by a `DEBUG` env var.
- **Config comes from env vars through `internal/env` helpers** with defaults. No config files unless the domain needs one (YAML for camera triggers).
- **Graceful shutdown:** `signal.Notify` on `SIGINT`/`SIGTERM`. Run the server and background tickers in goroutines, stop them on signal, and close channels to end workers.
- **Initialize empty slices** (`make([]T, 0)`) for anything JSON-encoded so it marshals as `[]`, not `null`.
- **Use `json:"snake_case"` struct tags**, and `json:"-"` for internal fields like file paths.

### Go testing

The existing Go projects have no tests yet; securitron tests manually against a running stack. When adding tests (recommended for domain packages like `media/`):

- **Table-driven tests** in `_test.go` beside the code, using the stdlib `testing` package and `httptest` for handlers. Run them with `go test ./...`.
- Use `t.TempDir()` for filesystem tests. Avoid assertion libraries unless the project already uses one.

---

## Containers, compose, and CI

- **Every service runs in Docker.** `compose.yaml` (or `docker-compose.yml` where it already exists) is how it's run locally and deployed.
- **Python image:** `ghcr.io/astral-sh/uv:python3.13-bookworm-slim`.
  - Copy `pyproject.toml uv.lock` first and `uv sync --frozen --no-dev --no-install-project` so the layer caches, then copy the code.
  - Set `UV_COMPILE_BYTECODE=1`, `UV_LINK_MODE=copy`, `PYTHONUNBUFFERED=1`.
- **Go image:** a multi-stage build. Build the UI (`npm ci && npm run build`), then the Go binary, then copy only the binary and runtime deps into a slim final stage.
- **Run as a non-root user** (`useradd` + `USER`). Add a `HEALTHCHECK` against the health endpoint. Put a header comment in the Dockerfile with the run command and URL.
- **Bind-mount permissions:** pass `HOST_UID`/`HOST_GID` via `.env` (not `UID`/`GID`, which bash reserves). Remap the app user in an entrypoint with `usermod`/`gosu`.
- **Compose:** `restart: unless-stopped`, `env_file: [{path: .env, required: false}]`, ports parameterized with defaults (`"${KESTREL_PORT:-8100}:8000"`). Pin the project name when compose lives in a subdirectory.
- **Cron-style jobs** use supercronic with a bind-mounted crontab (`supercronic -inotify /crontab`), so schedule edits need no rebuild. Keep a tracked `crontab.template` and gitignore the personal `crontab`.
- **`.dockerignore`** excludes `.git`, `.venv`, `.env`, caches, `node_modules`, and tests.
- **CI (GitHub Actions)** builds on push to `main`:
  - Services: multi-arch (`linux/amd64,linux/arm64`) images pushed to `ghcr.io/${{ github.repository }}`, tagged with the short sha and `latest`.
  - Skill CLIs: a date-versioned release (`vYYYY.MM.DD.HHMM`) zipping `SKILL.md` and `scripts/`.

`.gitignore` baseline, grouped with comments:

```gitignore
# python
__pycache__/
*.pyc
.venv/
.pytest_cache/
.ruff_cache/

# node
node_modules/
dist/

# local configuration
.env

# junk
.DS_Store
```

---

## Documentation

There are three audiences, and each gets its own file. **Update them in the same commit as the code they describe.**

| File | Audience | Content |
|---|---|---|
| `README.md` | people | what it is (first line), features, requirements, install, usage/commands with examples, configuration, and canonical upstream docs |
| `SKILL.md` | agents *using* the tool | frontmatter (`name`, `description` listing every trigger, `compatibility`), exact commands, options, output fields to parse, workflows |
| `AGENTS.md` | agents *working on* the code | layout, commands, conventions, gotchas, commit rules (see [AGENTS.md](#agentsmd)) |

- **Docs live next to the code.** A large project gets a README per area (`api/README.md`, `api/app/routes/README.md`, `ui/src/editor/README.md`). The root holds only what spans areas. Read the README for the area you're changing before changing it.
- **README tables for layout** (`| Path | What's in it |`) and for options.
- **Document units, ids, timestamps, and error shapes** explicitly in API READMEs.
- **Record findings that contradict upstream docs** (hardware or vendor quirks) in the repo, and say which source wins.
- **Plans and scratch notes don't get committed.** Fold what's durable into the README/AGENTS.md and delete the rest.

---

## Before you commit

1. Run the project's run-after-changes commands:
   - Python: `uv run ruff check . && uv run ruff format . && uv run pytest`
   - TS: `npm run lint && npm run typecheck`
   - Go: `go fmt ./... && go vet ./... && go test ./...`
2. Update the README / SKILL.md / AGENTS.md sections this change touches.
3. Review `git diff --staged`: one logical change, no stray files, no secrets, no debug prints, no commented-out code.
4. Write a one-line, lowercase, imperative message. Don't credit an agent.
5. If there's more than one logical change, split the commits (`git add -p`).

## Canonical documentation

- [AGENTS.md standard](https://agents.md)
- [uv](https://docs.astral.sh/uv/) · [PEP 723 inline script metadata](https://peps.python.org/pep-0723/) · [ruff](https://docs.astral.sh/ruff/) · [pytest](https://docs.pytest.org/) · [FastAPI](https://fastapi.tiangolo.com/) · [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [Effective Go](https://go.dev/doc/effective_go) · [Go code review comments](https://go.dev/wiki/CodeReviewComments) · [net/http routing patterns](https://pkg.go.dev/net/http#hdr-Patterns) · [log/slog](https://pkg.go.dev/log/slog)
- [TypeScript](https://www.typescriptlang.org/docs/) · [Vite](https://vite.dev/guide/) · [React](https://react.dev/) · [oxlint](https://oxc.rs/docs/guide/usage/linter)
