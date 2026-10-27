# coding

Personal coding standards for Python, JavaScript/TypeScript, and Go — project layout, tooling, style, testing, containers, documentation, and git — plus a procedure for generating a tailored, standalone `AGENTS.md` for any project.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

## What it covers

- **Git** — atomic commits, one-line lowercase imperative messages, short PRs, and a hard rule that no commit or PR ever credits a coding agent, model, or harness.
- **AGENTS.md** — always `AGENTS.md`, never `CLAUDE.md`; migrating existing files; and a survey-then-write procedure with a template for per-project `AGENTS.md` files that carry these standards without the skill loaded.
- **Principles** — explicit over clever, descriptive names, comments that explain why, lowercase structured logs, one error shape per interface, config from env with a committed template.
- **Python** — uv, ruff, pytest; the single-file PEP 723 CLI archetype and the FastAPI/pydantic-settings service archetype; style and testing with fakes, `TestClient`, and VCR cassettes.
- **JavaScript / TypeScript** — no-build vanilla JS for small UIs; React + Vite + oxlint + strict TypeScript for apps; a typed `fetch` client and abortable data hooks.
- **Go** — `internal/` per domain, thin handlers, structured `slog`, env helpers, embedded UI, graceful shutdown; recommendations beyond current practice are marked as such.
- **Containers and CI** — uv and multi-stage Go images, non-root users, healthchecks, compose conventions, and GHCR multi-arch builds.
- **Documentation** — README for people, SKILL.md for agents using a tool, AGENTS.md for agents working on it; docs live next to code and change with it.

## Generating a project AGENTS.md

With the skill installed, ask the agent to "write an AGENTS.md for this project". It surveys the repo, finds the real commands, keeps only the sections that apply, and copies in the git rules, so later sessions in that project follow the standards without loading this skill.

## Requirements

None. Applying the standards assumes the usual toolchains: [uv](https://docs.astral.sh/uv/) for Python, [Go](https://go.dev/dl/), and Node/npm for TypeScript. Claude Code reads `AGENTS.md` from v2.1.277, and only when no `CLAUDE.md` exists in or above the project.

## Canonical documentation

- [AGENTS.md standard](https://agents.md)
- [uv](https://docs.astral.sh/uv/) · [ruff](https://docs.astral.sh/ruff/) · [pytest](https://docs.pytest.org/) · [FastAPI](https://fastapi.tiangolo.com/)
- [Effective Go](https://go.dev/doc/effective_go) · [log/slog](https://pkg.go.dev/log/slog)
- [TypeScript](https://www.typescriptlang.org/docs/) · [Vite](https://vite.dev/guide/) · [oxlint](https://oxc.rs/docs/guide/usage/linter)
