---
plan_id: 001-persistent-tasks-and-web-task-board
spec: 001-persistent-tasks-and-web-task-board
feature: Persistent tasks in Postgres and a web task board to list and create them (dev)
gitops_app: ika100-demo/taskboard17-gitops
status: draft
repos:
  - id: taskboard17-gitops
    shape: gitops-app
    summary: Postgres database for the Task API, 2 API instances in dev, web wired to the API
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.5, AC-001.7]
    order: before-merge
    gitops:
      - {resource: postgres, type: database}
      - {uses: postgres, component: taskboard17-api}
      - {replicas: 2, component: taskboard17-api, instance: dev}
      - {env: {TASKBOARD_API_URL: 'http://taskboard17-api'}, component: taskboard17-web}
    depends_on: []
    done: false
  - id: taskboard17-api
    shape: service-python
    summary: Store tasks in Postgres so they survive restarts and are shared by every instance
    acs: [AC-001.1, AC-001.2, AC-001.7]
    criteria:
      - "Given the API starts against an empty Postgres database (`DATABASE_URL`), when a client sends `GET /tasks`, then the response is `200` with `[]`, and starting a second instance against the same database at the same time succeeds without error (system AC-001.1)"
      - "Given tasks A, B and C were created through `POST /tasks` in that order, when every API instance is restarted or replaced, then `GET /tasks` returns `200` with exactly A, B and C, oldest first, with the same `id`, `title` and `due_date` the create responses returned (system AC-001.1) (replaces AC-001.7)"
      - "Given two API instances use the same database, when a task is created with `POST /tasks` on one instance, then every following `GET /tasks` on either instance lists it, in creation order with tasks created on the other instance (system AC-001.2)"
      - "Given the database is unreachable, when a client sends `GET /tasks` or a valid `POST /tasks`, then the response is `503` with JSON body `{\"detail\": \"Database unavailable\"}` within 5 seconds, and once the database is back `GET /tasks` does not list the task of the failed request (system AC-001.7)"
      - "Given the database is unreachable, when the platform calls `GET /ready`, then it answers `503` with `{\"status\": \"not ready\"}` while `GET /health` keeps answering `200`; when the database is reachable again, `GET /ready` answers `200` with `{\"status\": \"ready\"}` without restarting the API (system AC-001.7)"
    arguments: |
      Connection comes only from the env the platform injects (DATABASE_URL, PG*); no other new env var.
      Order by a database sequence (insertion order), never by the opaque id or a timestamp, so creation
      order holds across instances. Schema creation/migration must be idempotent and safe when two
      instances start at once. The task model, routes, validation and 422 bodies of spec 001 stay as they are.
      Smoke tests write `smoke-<uuid>` tasks that now persist in the dev database; that is expected.
    depends_on: []
    done: false
  - id: taskboard17-web
    shape: web-nextjs
    summary: Web task board on the start page that lists tasks and creates a task with an optional due date
    acs: [AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.8]
    criteria:
      - "Given the Task API's `GET /tasks` returns tasks A, B and C in that order (C with `due_date` `null`), when a user opens the start page `/`, then the page lists exactly A, B and C in that order, each with its title and its due date as `YYYY-MM-DD`, and C shows \"No due date\" (system AC-001.3)"
      - "Given the Task API's `GET /tasks` returns `[]`, when a user opens `/`, then the page shows \"No tasks yet\" and the create form with a title field and a date input for the due date (system AC-001.4)"
      - "Given the board is open, when the user enters the title \"Buy milk\" and the due date 2026-12-01 (or leaves the due date empty) and submits, then the web sends `POST /tasks` with `{\"title\": \"Buy milk\", \"due_date\": \"2026-12-01\"}` (or `\"due_date\": null`), and on `201` the returned task appears at the end of the list without a page reload, the \"No tasks yet\" message is gone and the form is cleared (system AC-001.5)"
      - "Given the board is open, when the user submits a title that is empty, only whitespace, or longer than 200 characters after trimming, then the page shows \"Title must be 1 to 200 characters\" next to the title field, keeps the entered title and due date, and sends no request to the Task API (system AC-001.6)"
      - "Given the Task API is unreachable, does not answer within 5 seconds, or answers `GET /tasks` with anything but `200` and a JSON array, when a user opens `/`, then the page shows \"Tasks could not be loaded. Try again.\" and shows neither a task list nor \"No tasks yet\" (system AC-001.8)"
      - "Given the board is open, when the user submits a valid form and the Task API is unreachable, does not answer within 5 seconds, or answers anything but `201`, then the page shows \"Task could not be saved. Try again.\", keeps the entered title and due date, and the list does not show the unsaved task (system AC-001.8)"
      - "Given the user opens or reloads `/`, when tasks were created elsewhere since the last load, then the page shows the list as `GET /tasks` returns it at that moment, never a list cached at build time or from an earlier request (system AC-001.3, AC-001.8)"
    arguments: |
      The Task API base URL is the server-only env var TASKBOARD_API_URL (set by the gitops entry; document it in
      docs/env-vars.md, never NEXT_PUBLIC_). The browser never calls the Task API directly: the start page loads the
      list on the server and the create form submits through a Server Action (or a route handler of this app) that
      calls `POST /tasks`. No polling (spec A9). Tests stub the Task API at TASKBOARD_API_URL.
      In dev, smoke tests of both repos leave tasks in the database, so a deployed "No tasks yet" check cannot be a smoke test;
      keep a read-only smoke check that `/` loads and shows either the list or "No tasks yet".
    depends_on: []
    done: false
gitops_pin: []
---

## Decomposition

Three entries, built in parallel once the gitops entry is applied: the gitops-app entry (data: a Postgres database used by the API, 2 API replicas in dev, the web's API URL), the API (persistence, `503` on database outage, readiness), and the web (the board). No new repo and no shared library: the web only talks to the API over HTTP, and the contract below is all both sides share. The task model, routes and validation stay as the API's spec 001 defines them (system spec A3), so the web builds against the existing API and needs no `depends_on`.

## taskboard17-gitops — Postgres database for the Task API, 2 API instances in dev, web wired to the API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.5, AC-001.7 · **Depends on:** []

Declares the `postgres` database resource (one per environment, used only by the API, spec A1) and gives `taskboard17-api` its connection (`DATABASE_URL`, `PG*`), so tasks survive restarts (AC-001.1) and are shared (AC-001.2). AC-001.2 runs the API with 2 instances; the approved spec decided that value, so it ships here as `replicas: 2` in dev. The web gets `TASKBOARD_API_URL=http://taskboard17-api` (the API's in-cluster Service, port 80), which the board needs to list (AC-001.3) and create (AC-001.5). AC-001.7's readiness behaviour uses the existing `readiness: /ready` probe; no probe change. Applied before the components merge: until the API's persistence lands, the 2 in-memory replicas may show different lists in dev, which is acceptable for the short window (dev only, spec A2 drops in-memory tasks anyway).

## taskboard17-api — Store tasks in Postgres so they survive restarts and are shared by every instance

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.7 · **Depends on:** []

Replaces the in-memory store with Postgres behind the unchanged `POST /tasks` / `GET /tasks` contract. Owns durability and order across restarts (AC-001.1), consistency across instances (AC-001.2), and the outage behaviour: `503` on the task routes, readiness failing while liveness stays green (AC-001.7, spec A7). Its own criterion AC-001.7 ("restart returns `[]`") is superseded.

## taskboard17-web — Web task board on the start page that lists tasks and creates a task with an optional due date

**Shape:** `web-nextjs` · **Criteria:** AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.8 · **Depends on:** []

The first feature of the web: one start page `/` with the list and the create form below it (spec A4), browser date input (A5), new task appended at the end (A6), no auto refresh (A9), and the exact English wording of AC-001.4, AC-001.6 and AC-001.8 (A8). It owns the page side of AC-001.3 to AC-001.6 and all of AC-001.8.

## Contract

Caller: `taskboard17-web` (server side only). Provider: `taskboard17-api`, base URL `TASKBOARD_API_URL` (`http://taskboard17-api` in-cluster). JSON over HTTP, `Content-Type: application/json`. No authentication (open access in dev, see the spec's non-goals). Unknown extra fields in requests are ignored; callers ignore unknown extra fields in responses.

**Task object:** `{"id": string, "title": string, "due_date": string | null}`. `id` is server generated and opaque (do not parse it); `title` is stored trimmed, 1 to 200 characters; `due_date` is a calendar date `YYYY-MM-DD` or `null`. No other fields.

**`GET /tasks`** — list every task.
- `200`: JSON array of task objects, all tasks, in creation order, oldest first, no paging. `[]` when none.
- `503`: database unavailable (body below).

**`POST /tasks`** — create a task.
- Request: `{"title": string, "due_date": string | null}`; `due_date` may be omitted (same as `null`). The web sends `due_date: null` when the date field is empty, and the trimmed title it validated (1 to 200 characters).
- `201`: the created task object.
- `422`: validation error (title missing, not a string, empty or whitespace only, longer than 200 after trimming; `due_date` not a `YYYY-MM-DD` calendar date). No task is created.
- `503`: database unavailable. No task is created.

**`GET /ready`** (platform readiness probe): `200 {"status": "ready"}` when the database answers; `503 {"status": "not ready"}` otherwise. **`GET /health`** (liveness): always `200 {"status": "ok"}` while the process runs, never checks the database.

### Errors

- `422` uses FastAPI's standard body: `{"detail": [{"loc": ["body", "<field>"], "msg": string, "type": string, ...}]}`; the field is named by the last element of `loc` (`title` or `due_date`).
- `503` body: `{"detail": "Database unavailable"}`.
- The web treats every response other than `200` with a JSON array (for `GET /tasks`) or `201` with a task object (for `POST /tasks`) as a failure, including `422`, `503`, any other status, a non-JSON body and a connection error: it shows "Tasks could not be loaded. Try again." or "Task could not be saved. Try again.", keeps the form values and never renders a partial or stale list as current. (A `422` cannot normally happen, because the web checks the title rule itself and the date input only yields real dates.)
- The web never retries `POST /tasks` automatically (it is not idempotent); the user resubmits.

### Timeouts

- Web → API, `GET /tasks` and `POST /tasks`: 5 s total per call (connect + response), no automatic retry. On timeout the web shows the error message of that action.
- API → Postgres: connect timeout 2 s, query (statement) timeout 2 s; on either, or when no connection can be had, the task route answers `503` (well inside the web's 5 s).
- `GET /ready`: database check bounded to 500 ms so it answers within the probe's 1 s timeout; a slow or down database means `503` and the platform stops routing to that instance until it answers again. `GET /health` does not touch the database.

## gitops-app PR

One PR from the gitops entry, applied with `cplat` by `/system:ship` before the component PRs merge: the `postgres` database resource, `uses: postgres` on `taskboard17-api`, `replicas: 2` for the API in dev, and `TASKBOARD_API_URL` on `taskboard17-web`.

No pins: dev tracks `main`, so both components reach dev when their PRs merge, and promoting to staging or prod is a non-goal of this spec (both components have only a `dev` instance; a login spec comes first). `gitops_pin` is therefore empty.

After Argo reconciles dev, verify: the database cluster is healthy and the API pods have `DATABASE_URL`; 2 `taskboard17-api` pods are ready; `POST /tasks` then `GET /tasks` on `taskboard17-api.taskboard17-dev.localhost` lists the task, also after `kubectl rollout restart` of the API; `taskboard17-web.taskboard17-dev.localhost` shows the list and creates a task; both smoke suites pass.
