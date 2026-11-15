---
spec_id: 001-persistent-tasks-and-web-task-board
title: Persistent tasks and web task board
status: approved
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks and web task board

## Problem

The Task API keeps tasks in memory, so every restart or redeploy of the API loses all tasks, and several API instances would each see different tasks. The web component has no features yet, so the only way to see or create a task is to call the API by hand. People who want to use the task board today cannot rely on it to remember anything, and cannot use it without an HTTP client. With the API and the web component both deployed, this is the moment to make tasks durable and give people a page to work with them.

## Stories

As a task board user, I want the tasks I create to still be there after the system restarts or is redeployed, so that I can rely on the board as my record of what has to be done.
**Repos:** taskboard17-gitops, taskboard17-api

As a task board user, I want to open the web task board and see all tasks with their due dates, so that I know what has to be done without calling the API by hand.
**Repos:** taskboard17-web, taskboard17-api, taskboard17-gitops

As a task board user, I want to create a task with a title and an optional due date from the web task board, so that I can record work without an HTTP client.
**Repos:** taskboard17-web, taskboard17-api, taskboard17-gitops

## Acceptance criteria

- **AC-001.1** Given tasks were created through `POST /tasks` or the web task board, when every instance of the API is restarted or redeployed, then `GET /tasks` and the web task board show exactly the same tasks, with the same `id`, `title` and `due_date`, in the same order as before.
- **AC-001.2** Given the API runs with 2 instances in an environment, when a task is created through one request, then every following `GET /tasks` lists it, whichever instance answers.
- **AC-001.3** Given tasks A, B and C exist (C without a due date), when a user opens the web task board, then the page lists exactly A, B and C in creation order, oldest first, each showing its title and its due date as `YYYY-MM-DD`, with C showing "No due date".
- **AC-001.4** Given no task exists, when a user opens the web task board, then the page shows the message "No tasks yet" and the create form.
- **AC-001.5** Given the web task board is open, when the user enters the title "Buy milk" and the due date 2026-12-01 (or leaves the due date empty) and submits the form, then the new task appears at the end of the list without the user reloading the page, the form is cleared, and `GET /tasks` lists the task with `title` `"Buy milk"` and `due_date` `"2026-12-01"` (or `null`).
- **AC-001.6** Given the web task board is open, when the user submits a title that is empty, only whitespace or longer than 200 characters, then the page shows "Title must be 1 to 200 characters" next to the title field, keeps the entered values, and no task is created.
- **AC-001.7** Given the database is unreachable, when a client sends `GET /tasks` or `POST /tasks`, then the API answers `503` with a JSON error body, no task is created, and the API's readiness check reports not ready until the database is reachable again.
- **AC-001.8** Given the API cannot list or save tasks (it is unreachable or answers an error), when a user opens the web task board or submits the create form, then the page shows "Tasks could not be loaded. Try again." or "Task could not be saved. Try again." respectively, keeps the entered form values, and shows no partial or stale list as if it were current.

## Non-goals

- Reading, editing, completing or deleting a single task, in the API or the web task board.
- Filtering, searching, sorting options or pagination of the list.
- Logins, users, access control or per-user task lists: in dev anyone who can reach the environment may see and create every task, on the web task board and through the API, as the API allows today. A login spec comes before the board or the API is promoted to staging or prod.
- Database backups, restore and point-in-time recovery (a later spec).
- Moving tasks that exist in the API's memory at the time of the switch into the database.
- Promoting the database or the web task board to staging or prod; this spec covers dev.

## Assumptions

- **A1** One Postgres database per environment, declared as a resource of the system and used only by the API; the web component never reads the database directly (other option: a database shared with other systems; affects AC-001.1, AC-001.2, AC-001.7).
- **A2** Tasks that live in the API's memory when the switch happens are not carried over; the board starts empty in each environment, because in-memory tasks are lost on every restart anyway (other option: a one-off copy; affects AC-001.1, AC-001.4).
- **A3** The task model and API contract stay as they are: fields `id`, `title`, `due_date`, routes `POST /tasks` and `GET /tasks`, the same validation and `422` errors, creation order oldest first, full list without limit (other option: add `created_at` or paging now; affects AC-001.1, AC-001.3, AC-001.5).
- **A4** The web task board is the web component's start page, one page holding the list and the create form below it (other option: separate list and create pages, or a `/tasks` path; affects AC-001.3 to AC-001.6, AC-001.8).
- **A5** The due date is entered with the browser's date input, so only real calendar dates can be submitted; the web page checks the title rule itself (1 to 200 characters after trimming, same rule as the API) before sending (other option: free text date with API-side errors shown; affects AC-001.5, AC-001.6).
- **A6** After a successful create the new task is shown at the end of the list (creation order) rather than at the top (other option: newest first on the board only; affects AC-001.3, AC-001.5).
- **A7** A database outage answers `503` with a JSON error body on the task routes, and the readiness check fails so the platform stops routing to the instance; health (liveness) stays green so the API is not restarted in a loop (other option: `500`; affects AC-001.7).
- **A8** Error and empty-state wording is English and exactly as written in AC-001.4, AC-001.6 and AC-001.8 (other option: other wording; affects those criteria).
- **A9** The board is not refreshed automatically; tasks created elsewhere appear after the user reloads the page (other option: polling every N seconds; affects AC-001.3).

## Open questions

- ~~Who may see and create tasks? The web task board and the API would both be reachable by anyone who can reach the environment, with no login, and the board would show every task to every visitor. (suggested: yes, open access in dev as the API is today; logins come in a later spec before staging or prod; affects AC-001.3, AC-001.5, new criterion if access is restricted)~~ Answered 2026-11-15: open access in dev, as the API allows today; a login spec comes before staging or prod (see Non-goals).

## Changelog

- 2026-11-15 created
- 2026-11-15 amended: access question answered (open access in dev; login spec before staging or prod), folded into Non-goals
- 2026-11-15 approved
