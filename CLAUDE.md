# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project

**SnipVault** — a self-hosted code snippet manager. Backend-first portfolio project. See `project_description.md` for the full spec, milestones, and non-goals. If a request conflicts with `project_description.md`, flag it before proceeding.

**Positioning matters.** This project exists to demonstrate *software engineering* skills — REST APIs, databases, caching, concurrency, deployment, observability. It is deliberately not an ML project. Do not suggest ML features (language classifiers, semantic search embeddings, LLM-powered anything) even if they'd be technically interesting. If the user asks for one, push back once and confirm before implementing.

---

## Stack

- **Language**: Python 3.12
- **Web framework**: FastAPI (async)
- **ORM**: SQLAlchemy 2.x (async, `select()` style — no legacy `Query`)
- **Migrations**: Alembic
- **DB**: PostgreSQL 16
- **Cache / broker**: Redis 7
- **Background jobs**: Celery + Celery Beat
- **Server**: uvicorn (dev), gunicorn+uvicorn workers (prod)
- **Auth**: PyJWT (tokens), argon2-cffi (password hashing)
- **Config**: pydantic-settings
- **Tests**: pytest, pytest-asyncio, pytest-cov, testcontainers-python, Hypothesis (property-based), httpx (async API client), locust (load)
- **Lint / format**: Ruff (both linter and formatter — no Black)
- **Typecheck**: mypy in strict mode on `app/`
- **Container**: Docker, docker-compose for local
- **Infra**: Terraform for AWS, kustomize for k8s manifests
- **Observability**: prometheus-client, structlog, Grafana

Package management: **uv**. Use `uv add` / `uv sync`, not `pip install`.

---

## Repository Layout

```
app/
├── api/         # FastAPI routers — thin, delegate to services
├── core/        # config, security, logging, middleware
├── db/          # SQLAlchemy models, session, Alembic env + migration versions
├── services/    # business logic — all DB access flows through here
├── ds/          # hand-rolled data structures (trie, lru, token_bucket)
├── workers/     # Celery tasks
└── main.py      # app factory + router registration
tests/{unit,integration,load}/
infra/{terraform,k8s,nginx}/
ops/grafana/
docs/
```

**Layering rule:** `api → services → db`. Routers never touch SQLAlchemy directly. Services never import from `api/`. `ds/` is pure — no framework imports, no I/O.

---

## Conventions

### Python style
- Ruff config in `pyproject.toml`. Do not disable rules inline unless the reason is written in a comment on the same line.
- Type hints on every function signature. `mypy --strict` must pass.
- No `Any` in `app/` code. Use `object` or a `TypedDict` / `Protocol` instead. `Any` is allowed in tests.
- Prefer `from __future__ import annotations` at the top of every module.
- No wildcard imports.
- Docstrings on public functions in `services/` and `ds/`. Google style. Skip docstrings on obvious CRUD methods in routers.

### Async
- Everything in `api/` and `services/` is `async`. Blocking calls (CPU work, sync libs) go through `asyncio.to_thread` or into a Celery task.
- Never mix sync and async SQLAlchemy sessions. All app code uses `AsyncSession`.
- Do not call `.get()` or `.wait()` on a Celery `AsyncResult` from request handlers — enqueue and return a job ID.
- Celery tasks are sync functions. A task is a thin wrapper that calls `asyncio.run()` on an async service function, so business logic stays in `services/` and still uses `AsyncSession`. Workers create their async engine with `poolclass=NullPool`: pooled async connections are bound to the event loop that created them, and each `asyncio.run()` starts a new loop.

### Database
- All schema changes go through Alembic. Never edit a migration that has been merged to `main`; write a new one. A migration still on an unmerged branch may be regenerated.
- Every migration must be reversible (`downgrade()` implemented) unless it's a data-only migration explicitly marked non-reversible in the message.
- No raw SQL in `services/`. Use SQLAlchemy `select()` / `insert()` / `update()`. Exception: the view-count increment uses `UPDATE ... SET views = views + 1` via `sqlalchemy.update()` — this is intentional, do not "fix" it to a SELECT-then-UPDATE.
- Cursor-based pagination only. Never offset-based.
- Indexes for every new query pattern. If adding a query, add its supporting index in the same PR.

### API
- All request/response models are Pydantic v2, defined in `app/api/schemas/`.
- Errors follow RFC 7807 (`application/problem+json`). Use the `problem_response()` helper in `app/core/errors.py`.
- Every route has an OpenAPI `summary` and `response_model`.
- Version prefix is `/v1`. Do not introduce `/v2` without discussion.
- New endpoints require: router entry, Pydantic schemas, service function, at least one integration test, and an entry in `docs/api-changelog.md`.

### Auth
- JWT with short-lived access token (15 min) + refresh token (7 days).
- Passwords hashed with `argon2-cffi`. Never bcrypt, never plain SHA.
- Secrets loaded from env via `pydantic-settings`. Never hardcoded, never logged.
- The `get_current_user` dependency is the only path to reading the authed user. Do not decode JWTs elsewhere.

### Data structures (`app/ds/`)
- Pure Python, no external deps beyond stdlib.
- 100% unit test coverage. Property-based tests with Hypothesis where it fits (trie, LRU).
- Include a docstring section `Complexity:` stating time and space for each public method.
- These are showpieces. Code quality bar is higher here than elsewhere.

### Testing
- Coverage gate: **85% overall, 100% on `app/ds/`**. Do not lower these to make a PR pass.
- Unit tests are fast (< 100ms each) and hit no I/O.
- Integration tests use testcontainers or `docker-compose.test.yml`. Never mock the DB in integration tests.
- Every bug fix starts with a failing test that reproduces the bug.
- Don't test framework internals. Don't test Pydantic validation of trivial fields.

### Logging
- `structlog` configured in `app/core/logging.py`. Use `log = structlog.get_logger(__name__)`.
- No `print()`. No bare `logging.info(...)`.
- Every log line in a request context includes `request_id` and `user_id` (if authed) automatically via middleware — don't pass them manually.
- Log levels: `debug` for tracing, `info` for state changes, `warning` for degraded behavior, `error` for handled failures, `critical` reserved for "wake someone up."
- Never log secrets, tokens, passwords, or full request bodies.

### Metrics
- Use the `prometheus_client` registry in `app/core/metrics.py`. Do not create parallel registries.
- New metrics need a name following `snipvault_<subsystem>_<what>_<unit>`. Document in `docs/metrics.md`.

---

## Commit & PR Discipline

- **Conventional Commits.** Types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `chore`, `ci`, `build`. Scope in parens: `feat(snippets): add tag filtering`.
- Commits are small and self-contained. If a commit touches more than ~200 lines outside of generated files or migrations, it should probably be split.
- PRs target `main`. Every PR gets a description with: **what changed**, **why**, **how tested**. Even solo PRs.
- CI must be green before merge: Ruff, mypy, pytest, coverage ≥ 85% overall and 100% on `app/ds/`, Docker build.
- Squash-merge PRs. Keep `main` linear.
- Tag releases as `vX.Y.Z` with a changelog entry in `CHANGELOG.md` (Keep a Changelog format).

---

## Working With Claude Code

### Before writing code
1. Read `project_description.md` for the milestone context.
2. Check `docs/` for an ADR or design note on the area you're touching.
3. If the change spans more than one module, sketch the plan in the PR description before writing code.

### When adding a feature
- Start with the schema (Pydantic + SQLAlchemy + migration).
- Then the service function with unit tests.
- Then the router and integration test.
- Then docs + changelog.
- One PR per milestone item where feasible.

### When fixing a bug
- Write the failing test first. Commit it. Then fix.
- If the bug reveals a missing invariant, add it as an assertion or a schema constraint, not just a patch.

### When you're not sure
- Ask. Don't guess about business rules, auth policy, or data model changes.
- For style/naming ambiguities, follow the closest existing example in the same layer.
- If a task seems to require ML, an external SaaS, or a new top-level dependency, stop and confirm.

### What not to do
- Don't add dependencies casually. Every new dep needs a one-line justification in the PR description.
- Don't refactor unrelated code in a feature PR. Open a separate `refactor:` PR.
- Don't disable tests, lints, or type checks to unblock a merge. Fix the underlying issue or flag it.
- Don't introduce global state. Use FastAPI dependencies.
- Don't add `# type: ignore` without a reason comment.
- Don't write "TODO" without an issue link or a `# TODO(#123):` reference.

---

## Common Commands

```bash
# Setup
uv sync                              # install deps
docker compose up -d db redis        # start deps only
uv run alembic upgrade head          # apply migrations

# Dev loop
uv run uvicorn app.main:app --reload
uv run celery -A app.workers worker --loglevel=info
uv run celery -A app.workers beat --loglevel=info

# Quality gates (all must pass before PR)
uv run ruff check .
uv run ruff format --check .
uv run mypy app
uv run pytest --cov=app --cov-fail-under=85
uv run pytest tests/unit --cov=app/ds --cov-fail-under=100   # ds/ gate; the line above doesn't enforce it

# Running a subset of tests
uv run pytest tests/unit                                    # fast, no Docker needed
uv run pytest tests/integration                             # needs Docker running (testcontainers)
uv run pytest tests/unit/test_trie.py::test_prefix_search   # single test (example path)
uv run pytest -k lru -x                                     # match by name, stop at first failure

# Migrations
uv run alembic revision --autogenerate -m "add share_links"
uv run alembic downgrade -1

# Full stack local
docker compose up

# Load test
uv run locust -f tests/load/locustfile.py --host http://localhost:8000
```

The `uv run` commands above are the reference. `make` isn't installed on Windows by default, so if a `Makefile` is added, treat it as a shortcut, not the only way to run something.

**On Windows:**
- Celery's default `prefork` pool doesn't work on Windows. Run the worker with `--pool=solo` locally, or run it through `docker compose`.
- gunicorn only runs on Linux/macOS. It belongs in the prod container; use uvicorn on the host.

---

## Security Notes

- `.env` is gitignored. Commit `.env.example` with dummy values.
- Never commit secrets. If one leaks, rotate immediately and rewrite history.
- Rate limits are load-bearing, not decorative. Do not add unauthenticated endpoints without a rate limit.
- SQL injection: SQLAlchemy parameterization protects us — never build queries by string concatenation.
- CORS is off by default. Enable per-route only if needed.

---

## Documentation Expectations

Every merged milestone updates:
- `README.md` — quickstart, feature list, architecture diagram link.
- `CHANGELOG.md` — what changed.
- `docs/` — any new subsystem gets a short doc (goal: one page, not a book).

The README is the recruiter-facing surface. Keep it tight, with a clear "what this project demonstrates" section near the top.
