# Build Flowly

You are building **Flowly**: an open-source (AGPL-3.0), self-hostable planner that puts *today* from every
project and workspace on one board, with an AI concierge called **Flo** that knows your tasks, your calendar and
your team, and acts on them for you. This document is the complete specification. Build it incrementally,
following the milestones at the end; every feature below exists in the reference implementation and is described
with the rules that matter.

The reference code lives at `github.com/flowly-org`: `flowly-backend` (Django API + worker), `flowly-frontend`
(Angular web app, also packaged as Android/iOS with Capacitor), `flowly-desktop` (Tauri tray timer),
`flowly-org.github.io` (the static site).

---

## 1. Product

### 1.1 Who it's for

- Freelancers and small studios juggling clients, side projects and home in one day.
- Teams who want a shared plan without a heavyweight project tool.
- People who need fewer decisions per day, including people with ADHD (opt-in focus mode).
- Organizations that self-host and want their own AI provider, single sign-on and their data on their servers.

### 1.2 Principles

1. **One board for today.** "My day" shows everything assigned to you, from every workspace, in swimlanes. Planning
   is per day (`planned_date`); no date = backlog.
2. **The calendar is background, not tasks.** Events show above the board; only the few that need preparation are
   flagged (tickets, travel, a trip away from home).
3. **AI that does things, with review.** Flo uses tools to add, move, finish and plan tasks. Anything that creates
   many items at once (photo of a list, meeting notes, plan-my-week) is shown as a draft first and created only on
   confirmation. The delegation agent never sends, books or buys anything.
4. **Privacy.** Tasks are private / team / shareable / public. Private tasks are never broadcast to Slack, Discord,
   webhooks or team bots. Workspaces with their own AI endpoint are never sent to the platform model.
5. **Calm, elegant UI.** Line icons (24×24 stroke paths, one shared `app-icon` component), never emojis in the
   web UI (Telegram messages may use them). Light, dark and system themes; each workspace tints the logo, favicon
   and accents with its colour.
6. **Six languages**: English, Italian, Spanish, French, German, Portuguese, for every string in the UI and the
   built-in defaults (status and priority names are created in the user's language).
7. **Accessible and responsive**: keyboard reachable, aria labels on icon-only buttons, works from 390 px phones up.
8. **Self-hosting first.** One `docker compose up` gives everything. With plans off (the default) every feature is
   unlocked; a hosted instance can switch plans and Stripe billing on.

---

## 2. Architecture

### 2.1 Backend

- Python 3.12, **Django 5.2**, **Django REST Framework**, **PostgreSQL 16**, `djangorestframework-simplejwt`
  (access + refresh tokens), `django-filter`, `drf-spectacular` (`/api/schema/`, `/api/docs/`), `django-cors-headers`,
  `whitenoise`, `requests`, `cryptography`, `PyJWT`, `lxml`, `pypdf`, `icalendar`, `recurring-ical-events`,
  `django-storages[s3]` for attachments (local disk fallback: `ATTACHMENTS_DIR`, served through 5-minute signed links).
- One Django project `config` and apps: `accounts`, `workspaces`, `tasks`, `timetracking`, `integrations`,
  `concierge`, `calendars`, `delegation`, `wiki`, `donetick`, `imports`, `insights`, `agile`, `payments`,
  `mcpauth`, `mcpserver`, `sso`, `instance`.
- Served by gunicorn with **gthread** workers (3 workers × 8 threads, timeout 180 s): most requests wait on AI models
  or other APIs.
- A **worker process** (`manage.py run_worker`) runs a loop every few seconds: integration deliveries with retries,
  calendar sync, reminders, block starts, focus nudges, appointment briefings, morning briefs, weekly reviews, team
  bot jobs, standups/sprints, Donetick polling, delegation runs, wiki sync, Telegram polling and alerts. Each job is
  isolated (an exception in one never stops the others) and self-throttled (e.g. "at most once a minute").
- `docker-compose.yml`: `db` (postgres:16-alpine with healthcheck), `api` (migrates on start, then gunicorn),
  `worker` (same image, runs the loop). All configuration through environment variables (`.env`, with
  `.env.example` documenting them).
- **Instance settings** (`instance` app): every operational setting has a `Spec(key, group, label, help, type,
  secret, choices, public)`. Resolution order: database override (`InstanceSetting`, editable by staff in the Admin
  page; secrets encrypted) → environment / Django settings → default. Cached ~10 s per process. Public specs (e.g.
  Google client id) are served to the browser from `GET /api/config/`.

### 2.2 Web app

- **Angular** (latest), standalone components, **signals** (`signal`, `computed`, `effect`, `input`, `output`),
  new control flow (`@if`, `@for`), lazy-loaded routes, SCSS.
- Runtime config: `config.json` next to `index.html` (`{"apiUrl": "/api"}` or an absolute URL), loaded before
  bootstrap, so one build serves any backend. The native apps (Capacitor) can't use a relative URL and let the user
  pick one or more servers (§24).
- Core services: `ApiService` (typed methods, `run()` wrapper to promises, error → message), `AuthService` (tokens
  in localStorage, single in-flight refresh on 401), `WorkspaceStore` (workspaces, current workspace, projects,
  tags), `I18nService` + `t` pipe + `ldate` pipe, `ThemeService`, `ToastService`, `NotificationService` (polling,
  `version` signal that boards react to), `TimerService`, `FocusService`, `FloService` (panel state, `changed`
  counter that boards react to).
- Deployed to Netlify by GitHub Actions for the hosted instance; a Docker image (nginx serving the build, writing
  `config.json` from an `API_URL` env var at start) for self-hosting.

### 2.3 Other pieces

- **Mobile**: Capacitor wraps the same Angular build (`build:mobile`), Android APK built and signed in CI, iOS
  simulator build. Compact `/m` view by default on phones.
- **Desktop**: Tauri 2 tray app (Rust + plain HTML/JS window) for time tracking: running timer in the menu bar /
  tooltip, start a timer on today's tasks, untitled timer, open Flowly. Signs in with Flowly's OAuth (PKCE, loopback
  redirect); tokens stay in the Rust side, never in the web view. Supports several servers (§24).
- **Site**: static HTML/CSS on GitHub Pages (`flowly.net`), screenshots as webp, versioned stylesheet URL.
- **Releases**: Docker images on GHCR for backend and frontend; app installers attached to GitHub releases.

---

## 3. Accounts and authentication

- `User` (custom, email as username, no `username`): `email` (unique), `full_name`, `language` (`en|it|es|fr|de|pt`),
  `plan` (`free|pro|pro_plus`), `plan_expires_at`, `plan_source` (`manual|stripe|code`), `is_bot` (service users
  behind workspace tokens), `home_location` (free text, several separated by `;`), `github_login`, `adhd_mode`,
  `focus` (JSON options, §15), `ical_token`, `last_seen_at`.
- `has_pro` / `has_pro_plus`: true for staff, for everyone when plans are off, otherwise by plan and expiry.
- `POST /api/auth/register/` (email, password ≥ 8 with Django validators, full_name, language, optional invite
  `code`) → user + tokens; **onboarding** accepts pending invitations for that email, creates a "Personal"
  workspace, and links pending task shares sent to that email.
- `POST /api/auth/login/` (email + password → access/refresh). Refused with `{"detail", "sso": "required"}` when the
  email's organization requires SSO (§21), except for staff.
- `POST /api/auth/refresh/`, `GET/PATCH /api/auth/me/` (exposes `has_pro`, `has_pro_plus`, `plans_enabled`).
- **Personal API tokens** (`/api/auth/tokens/`): shown once, stored as SHA-256 hash with a short hint, `last_used_at`.
- **Last seen**: every authenticated request (JWT, API token, OAuth) and every Telegram message updates
  `last_seen_at`, at most every 5 minutes (one `UPDATE`, no read-modify-write).
- **Invite codes** (`InviteCode`: code `FLOWLY-XXXX-XXXX`, plan, days or forever, max uses, uses, valid until,
  note) and `CodeRedemption`. `GET /api/auth/code/?code=` previews; redeem at registration or
  `POST /api/auth/redeem/`. Redemption locks the code row (`select_for_update`). A code never downgrades a better
  current grant; Stripe sync never removes a forever code grant. Rate-limited (`invite_code` scope, 30/hour).

---

## 4. Workspaces

- `Workspace`: `name`, `color` (hex; palette cycles for a user's new workspaces), `time_tracking`, `plan`
  (`standard|enterprise`), `plan_expires_at`, `plan_source`, `created_by`. `is_enterprise` is true when plans are
  off, otherwise by plan and expiry.
- `Membership(workspace, user, role)`: roles `owner|admin|member`; `can_manage` = owner or admin. Unique per pair.
- `Invitation` by email (accepted on sign-up or by link) and **join links** (`/join/<token>`): preview public,
  accept requires sign-in; the role travels with the invitation.
- Every nested endpoint uses a mixin that resolves the membership from `workspace_pk` and answers **404 to
  non-members** (no id leaks); `require_manager()` raises 403.
- Creating a workspace creates the two system tag groups (§6) in the creator's language and makes them owner.
- Owners can delete a workspace (danger zone, with inline confirmation).

---

## 5. Projects

- `Project`: `workspace`, `name` (unique per workspace), `description`, `color`, `archived`, `time_tracking`,
  `position`. Projects are the default swimlanes.
- List annotates `open_task_count`. Reorder with up/down; archive/restore; delete keeps tasks (project set null).
- **Move to another workspace** (`POST …/projects/<id>/move/ {workspace, dry_run}`) and **Promote to a workspace**
  (`POST …/projects/<id>/promote/ {name, members: involved|all|me, dry_run}`). Managers only; target must be a
  workspace the user belongs to. `dry_run` returns a plan: tasks / open tasks, time entries, project integrations,
  tasks that become unassigned, links dropped, dependencies dropped, invoices staying; promote also returns people
  and unassigned counts per choice. Rules:
  - status / priority map by name (case-insensitive), else the equivalent (same "done" kind and rank); no new board
    columns are created in the target;
  - custom tags: same group and name, created in the target if missing;
  - assignees who aren't members of the target are unassigned;
  - project-level integrations, agile setup (with its sprints), billing, imports travel with the project; links to
    the source's workspace-wide integrations are dropped (external links deleted, Donetick links detached);
  - invoices stay in the source workspace, detached from the project;
  - dependencies on tasks that stay behind are removed; templates pointing at the project lose the reference;
  - name clash in the target → "Name (2)";
  - promote creates the workspace (named after the project, its colour), copies the source's statuses and priorities
    exactly, copies billing rates/terms (invoice numbering starts fresh), and adds the chosen people with their
    roles (source owner becomes admin; the caller owns the new workspace). The project stays a project inside it.
  - Everything runs in one transaction.

---

## 6. Tags, statuses, priorities

- `TagGroup(workspace, name, kind: custom|status|priority, position)`; one status and one priority group per
  workspace (partial unique constraint), which can't be deleted.
- `Tag(workspace, group, name, color, position, is_done)`; `is_done` only matters for status tags.
- Statuses are the board columns. Defaults: To do / In progress / Done (Done is done), priorities Low / Medium /
  High; names localized at creation. The **highest-positioned priority** means "urgent".
- Deleting a status moves its tasks to another status of the same done-ness (a workspace keeps at least one).
- Custom groups (e.g. "Context": @calls, @deep work) can become swimlanes; a task has at most one tag per group.

---

## 7. Tasks

### 7.1 Model

`Task`: `workspace`, `project`, `title` (255), `description`, `ref` (10-char public id from an unambiguous alphabet,
used in `/t/<ref>`), `visibility` (`private|team|shareable|public`), `status` (FK to a status tag, RESTRICT),
`priority` (FK, nullable), `planned_date` (null = backlog), `due_date`, `assignee`, `created_by`, `tags` (M2M),
`checklist` (JSON list `{id, text, done}`), `estimate_minutes`, `recurrence` (JSON rule), `series` (shared id of
occurrences), `sprint`, `position` (float, sort within a board cell), timestamps, `completed_at` (set/cleared when the
status' done-ness changes).

Related: `Comment` (author or AI `source`), `Attachment` (upload to storage, link, Google Drive, OneDrive /
SharePoint; download endpoints check access), `TaskShare` (by email; user linked on sign-up), `Reminder` (`at`
datetime, sent once), `TaskTemplate`, `TaskDependency(task, depends_on)` (no self, no cycles).

### 7.2 Visibility and access

- private: creator and assignee only · team: workspace members · shareable: team + people it's shared with (view and
  comment) · public: anyone with the link, read-only, no account.
- One reusable query (`member_visible_q(user)`) is used everywhere tasks are listed for a member, including search,
  Flo's tools and the MCP server.
- `/t/<ref>` page: the task, its comments (sharers can comment), attachments; the right actions per access level.

### 7.3 Recurrence

Rule `{freq: daily|weekly|monthly|yearly, interval 1–365, weekdays [0..6, Monday=0] for weekly, after_done: bool,
until: date}`; `{}`/null = none. When an occurrence is done, the next one is created (same series, checklist reset,
status back to the default open status, date from the rule or from the completion date if `after_done`), and takes
the rule over. Human description of the rule for the UI and for Flo.

### 7.4 Bulk add syntax

One task per line (`- ` optional). Tokens anywhere in the line:
`!high` priority (prefix match on the workspace's priorities) · `+Project` (exact or prefix, dashes for spaces) ·
`@today|@tomorrow|@monday…|@YYYY-MM-DD|@+N` (days from a start date, used by templates) · `~30m|~1h|~1h30` estimate ·
`=Name` assignee (member by name or email prefix) · `#tag` · `[x]` done.
Indented lines under a task: plain text → description; `[ ]` / `[x]` lines → checklist items.
`POST …/tasks/bulk/` previews (dry run) or creates. The bulk dialog shows a live preview with all parsed parts.

### 7.5 Templates

`TaskTemplate(workspace, project, name, body)` where body uses the bulk syntax with `@+N` offsets. Apply with a start
date; create from selected tasks (`from-tasks`). Flo's `use_template` tool.

### 7.6 Dependencies

`blocked_by` on task write (write-only) and in the representation; blocked tasks are marked (link icon) and listed in
the dialog; when the last blocker is done, the blocked task's assignee is notified. Cycles rejected.

### 7.7 Day load

`GET /api/my/day-load/?date=`: estimated minutes of my open tasks that day (plus carried-over today), tasks without
estimate, busy minutes from timed events (merged overlaps, **plus travel time**, §11.4), capacity
(`focus.capacity_minutes`, default 480), free and over minutes. Shown in the toolbar ("≈ 5h planned · 2h free ·
1h too much" in red when over).

### 7.8 Search

`GET /api/my/search/?q=` across visible tasks (title, description, comments), projects, templates and wiki pages,
ranked; opened with **Ctrl/Cmd+K** as a palette with type icons and keyboard navigation.

### 7.9 Custom attributes

- `Attribute(workspace, project?, key, value)`: free facts for Flo (a client's email, contract terms, tone, repo). A
  project's value overrides the workspace's for the same key. Keys are slugged (`client_email`), start with a letter,
  and can't be words bulk-add reads as dates (`today`, weekdays…). `/workspaces/<id>/attributes/` (`?project=`):
  members read, owners/admins write. Edited in workspace settings and per project (with inherited ones and
  "Override for this project").
- In a task, `{{key}}` is a **placeholder**: the stored text keeps it; the task API returns `title`/`description`
  filled in (titles flattened and capped at 120 chars per value) plus `title_template`/`description_template` with
  the raw text, which the edit form uses. Values load once per workspace per request.
- `@key` (not inside an email) **points Flo at** an attribute: the task API lists `mentions` [{key, value}], shown
  under the notes. Flo's task lines include the attributes a task uses; its workspace overview lists every
  workspace's and project's attributes; the delegation agent gets them as context; integrations send the filled-in
  text while change detection keeps the stored one. The task dialog offers chips for every usable attribute.

---

## 8. My day and the board

- Routes: `/day` (all workspaces, "global" mode) and `/w/<id>/board` (one workspace).
- Toolbar: previous / Today / next, date title and picker, progress bar (done/total), day load; then icon controls
  with hover labels: swimlanes select (project, workspace, assignee, priority, any custom tag group), "Add to"
  workspace (global, if several), Mine / Everyone segmented, toggles **Carry over** (unfinished earlier tasks),
  **Hide empty lanes**, **Backlog** panel (count badge), **Plan** (time blocking, §12); Templates, Bulk add, Simple
  view; primary **New task**.
- Columns = statuses; lanes collapsible; drag and drop (CDK) between cells and lanes updates status / lane value /
  date; drag from the backlog panel onto the board plans it for that day. Card: checkbox, title, badges (due with
  flag, blocked, checklist n/m, comments, attachments, estimate, recurrence, visibility icon, assignee initials),
  timer play button.
- Task dialog: tabs Details / Discussion / Files / Share; details include checklist, repeat picker, reminders,
  estimate, dependencies, delegate to AI (§16.7).
- Calendar strip above the board (§11), Flo's morning brief card and the plan-my-week card (§16), the ADHD "Now"
  card (§15).
- Boards reload automatically when a notification arrives or Flo changed something.
- **Simple view** (`/m`, default on phones, one click from My day on desktop): today's list with big play buttons,
  timer bar, photo of a list, quick add, and the **swipe deck**: cards one at a time, swipe right (or →, ✓) = done,
  left (or ←, ✕) = keep; undo; keyboard hints on desktop.

---

## 9. Time tracking, billing, invoices

- `TimeEntry(workspace, user, task, project, description, started_at, ended_at (null while running),
  duration_seconds, billable, invoice)`; **one running timer per user** (partial unique constraint).
  `GET/POST/DELETE /api/my/timer/` (start on a task or untitled; starting stops the previous). Workspace and project
  can switch tracking off.
- Time page: week view, entries list and edit, report by project/person (`time-entries/report/`).
- `WorkspaceBilling` (enabled, currency, default hourly rate, rounding minutes, billable by default, tax rate,
  invoice prefix, next number, payment terms, notes) and `ProjectBilling` (mode inherit / own hourly rate / not
  billable, client identity incl. Italian e-invoicing fields, client email and reference, budget hours).
- Invoices: draft from unbilled entries in a period and project (lines grouped, rounding applied), numbering per
  workspace, statuses draft/sent/paid/void, CSV export, **e-invoices**: FatturaPA (Italy), XRechnung and Factur-X
  (EN 16931 UBL/CII), validated against bundled XSDs; issuer identity from the user's `InvoicingProfile` (name,
  VAT id, tax code, address, IBAN, format-specific settings by country).
- **Pay online**: the user's own Stripe key (encrypted in their invoicing profile) creates a Payment Link for an
  invoice; paid status checked and set automatically.

---

## 10. Notifications

- `Notification(user, kind: assigned|urgent|comment|brief|shared|focus|reminder, title, body, workspace, task, data,
  read_at, telegram: none|pending|sent|failed)`.
- Rules: assigned to you by someone else; urgent (top priority) for today; new comments on your tasks; shares;
  reminders; Flo's nudges and briefings. Your own changes never notify you.
- Bell in the top bar (icons per kind), mark read, click opens the task. Telegram copies go out when the user linked
  Flo and keeps alerts on (Pro when plans are on); desktop notifications in the browser and tray app.

---

## 11. Calendars

### 11.1 Accounts and sync

- `CalendarAccount(user, provider: google|microsoft|caldav|ics, email, server_url, username, encrypted tokens or
  password, status, last_error, last_synced_at)`, `Calendar(account, external_id, name, color, primary, selected)`,
  `CalendarEvent(calendar, user, external_id, title, description, location, html_link, all_day, start, end,
  start_date, end_date, auto_reason, prep: auto|needed|background, confirmed_at, tasks M2M)`.
- Google and Microsoft via OAuth (several accounts each), **CalDAV** with an own small client (discovery through
  principal → calendar-home-set, REPORT for the window; app passwords; Nextcloud, iCloud, Fastmail, Radicale…),
  **iCal subscriptions** (any public or secret .ics/webcal URL, refreshed every 15 minutes, 5 MB cap). iCalendar
  parsing with recurring events expanded in the window; occurrences keyed by UID + start.
- Sync window: some days back, a few months ahead; events that disappear are deleted. Only selected calendars sync;
  your own calendars are selected by default, holidays/birthdays/others' are opt-in.
- **Feed out**: a private iCal URL with your planned tasks, due dates and time blocks (`/api/ical/<token>.ics`);
  regenerate or turn off.

### 11.2 Preparation heuristics

`prep.classify(title, location, description, home, days)` → reason or "": keyword lists in six languages for
`travel` (flights, trains, hotels, check-in…), `tickets` (concerts, theatre, museums…), `event` (conferences,
weddings, birthdays), else `away` when the address (with a comma) isn't in a home town, else a multi-day event with a
place = travel. Online locations (URLs, Meet, Zoom, Teams) never count. Home parsing drops the country, postcodes and
province codes. The user overrides per event (needs preparation / just background / automatic) and can confirm an
away event is still on. "To prepare" list for the next 3 weeks with linked tasks or keyword-related suggestions.

### 11.3 Day strip

Chips with time (or "all day"), title, place, preparation flag icon, leave-by time, linked-task count. Click opens a
detail: date/time range, place, leave-by line with a map link, preparation reason, linked tasks (unlink), related
tasks (link it), quick "add a preparation task" (title, workspace, date), confirm/background buttons, link to the
original event.

### 11.4 Travel time and leave-by

- Setting `focus.transfer_minutes` (default 45, 0–240). An event with an address **in a home town** takes that long
  to reach; one **in another town** takes twice that; online events, events without an address and all-day events
  need none.
- Travel counts as busy time **before and after** the event, in the day load and in time blocking; the event API
  returns `travel` (`in_town|away|""`), `travel_minutes`, `travel_source` (`driving|fixed|""`), `leave_at`. Flo's
  brief mentions when to leave.
- **Real driving time (opt-in, `focus.travel_mode = fixed|driving`)**: the worker geocodes the first home and each
  upcoming (14 days) in-town or away appointment with **Nominatim** and routes it with **OSRM**, storing
  `CalendarEvent.drive_minutes` with `drive_key` (hash of home + location) so a change recomputes it; pages never call
  the network. Travel = drive + 5 min parking, rounded up to 5; fixed estimate until known or when there's no route.
  Fair use of the public servers: ≤ 1 Nominatim request/s, a descriptive User-Agent, every address cached
  (`GeoPlace`, not-found retried after 7 days), ~12 routes per minute overall; outages aren't cached. Geocoding asks
  for 5 candidates biased to home (viewbox) and accepts only one in a **town named in the address** (city, town,
  village, suburb… never county/state: Italian provinces share their capital's name); otherwise "not found".
  Instance settings `NOMINATIM_URL`, `OSRM_URL`, `OSRM_PROFILE` (on-premises servers on private addresses allowed);
  admin check "Maps" geocodes and routes a test trip.

### 11.5 Appointment briefing

Opt-in setting `appointment_briefs: off|in_person|all` and `appointment_lead_minutes` (default 15). The worker,
once a minute, sends each due event once (dedup key per event and start): "<title> at HH:MM", date and time range,
leave-by line (out-of-town wording), address with a Google Maps search link, notes (300 chars), linked tasks with
done/still open, the original link. Delivered as a reminder notification and on Telegram whenever Flo is linked,
regardless of the alerts setting.

---

## 12. Time blocking

- `TimeBlock(user, task, title, start, end, reminded)`; working hours `focus.day_start/day_end`.
- Busy spans = selected timed events not marked background (widened by travel) + existing blocks, merged. Free slots
  start on the quarter hour, from now if today.
- `GET /api/my/blocks/?date=` (blocks, free slots, events), CRUD, `POST /api/my/blocks/plan/ {date, apply}`:
  **auto-plan** fits the day's open tasks by priority and estimate (30 min when unknown) into free slots; preview
  first, apply creates blocks.
- Day-plan timeline: hours grid, events in grey, blocks draggable/resizable (click vs drag detection), overlapping
  blocks in lanes, queue of tasks to place. Block-start notifications (opt-out).

---

## 13. Weekly review and insights

- `WeeklyReview(user, week_start, stats, text)`: tasks done, minutes tracked, planned vs done, overdue, estimate vs
  real ratio, next week's load against capacity, time by project, done by project, "still open" list with one-click
  fixes (next Monday / backlog / done). Flo writes the review text every week at the user's chosen day and time (and
  on demand), with three priorities for next week.
- Insights for me / a project / a workspace (`/api/insights/?scope=&id=`): throughput per week, created vs done,
  time per week and per project, on-time rate, median time to done, estimates vs real time, open and overdue per
  person. Simple SVG bar charts (horizontal and vertical) with accessible labels.

---

## 14. Timeline and dependencies

`/w/<id>/timeline`: tasks as bars from planned date to due date grouped by project, milestones (due date only),
arrows for dependencies, today line, zoom; drag a bar to move both dates, drag the end to change the due date;
filters by project. Blocked tasks show a link icon.

---

## 15. Focus and ADHD mode

- `adhd_mode` toggles; `focus` options with validation and limits: `today_limit` (3), `check_in_minutes` (25),
  `hyperfocus_minutes` (90), `heads_up_minutes` (10), `celebrate`, `calm` (no motion), `capacity_minutes`,
  `day_start`, `day_end`, `block_reminders`, `transfer_minutes`, `appointment_briefs` (choice), and
  `appointment_lead_minutes`.
- "Now" card on My day: the single next task with Focus 25 min / Done / Break it down (Flo splits it into 3–6 tiny
  checklist steps) / Not now, plus the next two.
- Phone shows "Today's N" with the rest folded. Nudges from the worker (once a minute, dedup by key): check-in while
  a timer runs, break after long stretches, heads-up before events. Optional small celebration bursts.
- Flo's prompts adapt: shorter, concrete, no guilt.

---

## 16. Flo, the concierge

### 16.1 Profile and channels

`ConciergeProfile(user, enabled, name "Flo", language auto|code, timezone, brief_time, morning_brief, alerts,
review_day, review_time, personality, memory, learn, provider, telegram_chat_id, telegram_username, link code)`.
`ConciergeMessage(user, role, content, channel: telegram|web|brief|setup)` is one conversation across channels
(history of the last N messages goes into each turn).

- **Telegram**: link with a one-time code (`/start <code>` deep link), polling or webhook (secret header); commands
  (`/today`, `/apps`, help…), free text, **voice messages** (transcribed by an audio-capable model), **photos**
  (lists read by a vision model) and documents.
- **In-app panel**: "Ask Flo" pill in the top bar (Ctrl/Cmd+J) opens a right drawer with the conversation, suggested
  questions, a composer (Enter sends), "Plan my week", Telegram messages labelled. When Flo is unavailable (free plan
  with plans on, or no AI provider) the panel says why and still offers plan-my-week.

### 16.2 The loop

`run(user, messages)`: chat completion with tools (OpenAI-compatible function calling), up to **10 steps** and a
**70-second budget**; when either runs out, one last call without tools asks for a summary of what was done (never
"I got lost"). Tool errors are returned to the model as text. Each turn and its answer are stored.

### 16.3 Tools

`list_tasks` (by day or backlog; with a `query` and no day it searches open tasks on every date, all words must
match), `list_workspaces`, `create_task`, `add_tasks` (bulk syntax), `update_task` (title, day, status, priority,
estimate; moving never creates a copy), `set_repeat` (only when asked; `{}` stops), `complete_task`, `add_comment`,
`start_timer`, `stop_timer`, `list_events`, `create_event_task`, `link_event_task`, `mark_event`, `delegate_task`,
`set_dependency`, `block_time`, `plan_day`, `my_week`, `post_standup`, `use_template`, `set_reminder`, `add_steps`,
`search_wiki`, `read_wiki_page`, `remember`, `forget`, `app_downloads`. Tool descriptions are explicit about when
NOT to use a parameter (models over-fill optional fields).

### 16.4 Morning brief and memory

- Brief at the user's time (Telegram, and as a card on My day: fetched with `GET /api/concierge/brief/`, written on the
  first visit of the day if missing, dismissible until tomorrow): greeting, plan grouped by workspace/project,
  urgent and carried-over first, due today, calendar context (only where it shapes the plan, events needing
  preparation, leave-by times), day load, delegable tasks offered, a suggested first task. Falls back to a plain
  text brief if the model fails.
- Memory: Flo keeps short notes about the user (`remember`/`forget`, nightly reflection when `learn` is on).

### 16.5 Plan my week (onboarding)

Shown inline on My day when today and the backlog are empty, and as a dialog from the panel:
"What's on your plate this week?" → `POST /api/concierge/setup/ {text, workspace?}` returns a **proposal**
`{message, projects:[{name, exists}], tasks:[{title, project, date, minutes, priority, notes}]}`; the user removes
items and confirms with `{apply: true, plan}`. The model replies JSON only; the server normalizes everything
(projects reused case-insensitively, ≤ 6 projects, ≤ 30 tasks, dates only from today to +90 days, estimates
5–480, priority only "high", unused projects dropped). Works on every plan, 5 proposals a day per user; recorded
in the conversation so Flo remembers it.

### 16.6 Notes, photos, emails

`POST …/tasks/from-notes/` (meeting notes, transcripts, long emails → bulk-syntax action items with assignees by
name, dates stated or implied, context lines) and `…/from-photo/` (handwritten lists, done items recognised). Results
open in the bulk-add preview. Email-in integration can use the same extraction (`ai: off|auto|always`).

### 16.7 Delegation agent

`Delegation(task, user, run, status, verdict full|partial|no, plan, reason, summary, left_for_you, steps, tokens,
error)`. First an assessment (what it can do, what's left for you); then a run with research and drafting tools
(web search/fetch, wiki, the task's context), reporting on the task with sources. Never sends, books or buys.
Monthly cap on Pro Plus when plans are on; no cap when off. Flo offers delegable tasks in the brief.

### 16.8 Providers

Platform provider = OpenRouter-compatible base URL + key + default model (+ audio, vision, delegation, triage
models), with a per-user daily token cap. Workspaces can bring their own provider (OpenRouter key or any
OpenAI-compatible endpoint, e.g. on-premises); data from those workspaces then goes only there, and is hidden from
the platform model. `AIUsage` records tokens per user/day/provider.

---

## 17. Agile rituals (opt-in per workspace or project)

`AgileSetup(workspace, project?, enabled, sprint_weeks, sprint_start weekday, carry_over, standup, standup_time,
standup_days, retro, timezone)`; `Sprint(number, goal, start, end, status planned|active|closed, review)`;
`StandupEntry(sprint, user, date, yesterday, today, blockers)`; `RetroItem(sprint, kind went_well|improve|idea,
text, author, votes, task)`.
- Sprints page: goal, capacity vs committed estimates, burndown, sprint backlog, add/remove tasks; closing a sprint
  carries unfinished tasks over and opens the next (keeping the length).
- Async standup: drafted from each person's tasks, answered on the page or by replying to Flo; summary posted to the
  team at standup time.
- Retro board; items become tasks for the next sprint.

---

## 18. Integrations

Generic model: `Integration(workspace, project?, kind, name, enabled, config JSON, credentials JSON encrypted, token
(public id for incoming URLs), secret (signing))`; `ExternalLink(integration, task, external_id, url, data)`;
`Delivery(integration, task, direction in|out, event, payload, status pending|success|failed|ignored, attempts,
next_attempt_at, response_status, response_body, last_error)` doubling as the outgoing job queue (exponential
backoff, 4xx except 408/409/429 not retried, manual retry from the UI).

A connector registry: each kind defines `clean_config`, `outgoing`, `default_events`, `enterprise`, `wants`,
`deliver`, `test`, `info`. Task events: `task.created, task.updated, task.status_changed, task.completed,
task.assigned, task.deleted, comment.created` with a self-contained payload (task with workspace, project, status,
priority, assignee, tags, dates, ref, link) and a `changes` dict. A project-scoped integration only sees its
project's tasks. **Private tasks never reach broadcast kinds** (Slack, Discord, outgoing webhooks).

Kinds:
- **GitHub** (GitHub App created from a manifest by the instance admin; per-project repo): issues ⇄ tasks, closing
  either side closes the other, comments both ways, labels → tags, assignees matched by GitHub login; webhook
  signature checked.
- **Slack** (incoming webhook) and **Discord** (channel webhook URL, only `discord.com` webhook URLs accepted; rich
  embeds with colour per event, title link, assignee/due/priority fields, workspace · project footer; Discord
  markdown escaped and `allowed_mentions` empty so nobody is pinged).
- **Outgoing webhook**: signed JSON (`X-Flowly-Signature: sha256=HMAC`), secret rotation.
- **Incoming webhook**: secret URL that creates or updates tasks (status, default schedule backlog or today).
- **Email in**: a per-integration address and user aliases (`team@in.<domain>`), plain task or AI extraction of
  action items from long emails; inbound endpoint authenticated with a key.
- **Telegram group** and **Slack app** team bots (Enterprise): the team's plan every morning (time, weekends,
  timezone), live updates, questions answered in the chat, whiteboard photos → tasks, `/undo`; in a group, private
  tasks never appear, people outside the workspace can only read, everyone else acts as themselves.
- **Wiki**: a GitHub wiki, a docs folder in a repo, or a **MediaWiki** (API, namespaces), synced to `WikiPage(path,
  title, text)` for search; Flo and the delegation agent use it as team knowledge.
- **Donetick** (self-hosted chores): workspace/project integration or personal account; chores become tasks, done ⇄
  done, write-back of renames/dates, per-label routes to workspaces/projects, polling + webhook, outbox with retries.
- **Stairway** (an MCP server that runs coding agents in folders on a machine of yours; secret = its token): from
  the task dialog, "Work on it in Stairway" creates an experiment `flowly-<ref>` briefed with the task (title, notes,
  checklist, attributes, discussion, links, extra instructions; write files with file tools, leave results in
  ./deliverables, end with a summary). `StairwayRun(task, experiment, status working|waiting|review|failed|stopped,
  last_reply, waiting_for, attachment_ids)`. Stairway calls back signed (HMAC) on each reply (→ a comment; the
  worker zips ./deliverables into the task's attachments, replacing the previous batch, and notifies) and when the
  agent asks for permission (→ "Waiting for your OK" with the request, Allow/Refuse). Follow-ups, Stop, links to the
  session and to the folder in VS Code.
- **OpenRouter** / **OpenAI-compatible endpoint** (Enterprise): the workspace's own AI provider.
- **Imports**: Trello board JSON and Asana CSV or token + project link; lists/sections become tags, checklists,
  due dates, members by name/email, comments; re-imports add only what's new (`ImportRun.items`).

The Integrations page: kind cards (brand-coloured line icons; Enterprise badge), connected list with on/off toggles,
an editor per kind (name, applies to: whole workspace or a project, kind fields, event checkboxes), Send test,
deliveries log with retry, rotate secret, setup hints.

---

## 19. MCP server and OAuth

- MCP endpoint `/api/mcp/` (personal: all of the user's workspaces) and `/api/mcp/w/<id>/` (one workspace), JSON-RPC
  over streamable HTTP, tools to list/search/create/update/complete tasks, timers, events, time and invoices, with the
  caller's permissions.
- OAuth 2.1 authorization server for ChatGPT, Claude and other MCP clients: dynamic client registration, authorization
  code with PKCE, consent page in the web app (`/oauth/authorize`), refresh tokens, revocation, protected resource
  metadata and `/.well-known/oauth-authorization-server`; grants listed and revocable in Settings.
- Workspace **service tokens** (`/workspaces/<id>/mcp-tokens/`) act as a bot member for automations (n8n, Zapier).

---

## 20. Calendar, tasks and the AI on the phone and desktop

Covered in §2.3 and §24; the `/apps` page lists downloads per platform (detecting the visitor's), release notes and
checksums from the public releases repository. Flo can send download links (`app_downloads`).

---

## 21. Single sign-on (Enterprise workspaces)

- `SsoConnection(workspace one-to-one, issuer, client_id, client_secret encrypted, label, enabled, required, auto_join,
  tested_at, tested_by, last_error)`, `SsoDomain(connection, domain, token, verified_at)` (a verified domain belongs
  to one workspace: partial unique constraint), `SsoLogin(state, connection, nonce, verifier, next, test)`.
- OpenID Connect authorization code + PKCE (S256). Discovery from `<issuer>/.well-known/openid-configuration`
  (issuer must match; Entra's `{tenantid}` template handled). Client secret via HTTP Basic when supported, else in the
  body. ID token verified against the JWKS (asymmetric algorithms only), issuer, audience, expiry, nonce. Email from
  `email` (unless `email_verified` is false) or an email-shaped `preferred_username`/`upn`.
- Domains proven with a DNS TXT record `_flowly-verification.<domain>` = `flowly-verification=<token>` looked up over
  DNS-over-HTTPS; public mail domains (gmail.com, outlook.com…) refused.
- Endpoints: `GET /api/auth/sso/discover/?email=` · `POST /api/auth/sso/start/ {email, next}` → `{url}` (an
  authenticated workspace admin can pass `{workspace, test: true}` to test before enabling) · `GET
  /api/auth/sso/callback/` → redirect to the app's `/sso#access=…&refresh=…&next=…` (or `#error=`); the app stores
  the tokens and clears the fragment from history. Throttled.
- Sign-in creates the account on first use (unusable password, onboarding), links an existing account by email,
  joins the workspace as member when `auto_join`, refuses deactivated users and emails outside verified domains.
  A successful test by an admin sets `tested_at`.
- **Required** blocks password login and password sign-up for the domains; it needs a successful test; changing the
  provider resets the test and is refused while required; instance staff keep password access (break-glass); losing
  Enterprise switches SSO off.
- Login page: typing an email checks discovery (debounced); "Continue with <label>" appears; when required the password
  field disappears. Workspace settings: provider presets with setup hints (Google Workspace, Microsoft Entra ID,
  Okta, other OIDC), redirect URI to copy, domains with their TXT records and Verify, test, enable/require/auto-join
  switches, remove.

---

## 22. Plans and payments

- `PLANS_ENABLED` instance setting (env `PLANS_ENABLED=1`), **off by default**: every account is Pro Plus and every
  workspace Enterprise, payments are off, delegation has no monthly cap. `/api/auth/me/` exposes `plans_enabled` and
  the UI hides plan pickers, upgrade prompts and Enterprise badges' locks.
- On (hosted instance): Free / Pro (concierge, AI photo import, invoices with e-invoices and pay links) / Pro Plus
  (plus delegation, 60 runs/month) for people; Enterprise for workspaces (team bots, own AI provider, SSO), set by
  instance admins. Stripe: products and prices created by an admin "Set up" action (monthly/yearly, tax exclusive,
  Stripe Tax optional), Checkout, Customer Portal, webhook keeping `Subscription(kind, status, interval, amount,
  current_period_end, cancel_at_period_end)` in sync and the user's plan (never removing a forever code grant).

---

## 23. Instance administration (`/manage`, staff only)

- Overview: counts, failed and queued deliveries, Telegram users, AI usage today, **activation funnel** (signed up →
  created a task → talked to Flo → linked Telegram → active this week) with bars; service checks (Telegram bot,
  OpenRouter, file storage, GitHub App).
- Settings: every `Spec` grouped (General, AI, Telegram, Stripe, Documents, Email, GitHub, Microsoft…), secrets masked,
  source shown (admin / environment / unset), reset to environment.
- Users: search, plan, expiry, staff, active, workspaces, AI tokens (30 days), tasks created, Flo messages, Telegram
  linked, last seen, joined.
- Workspaces: plan, expiry, members. Invite codes: create, list, uses, delete.
- GitHub App: create from manifest or connect an existing one; Telegram webhook on/off; Stripe setup.

---

## 24. Apps and multiple servers

- **Web**: one instance per deployment (`config.json`).
- **Mobile (Capacitor)**: the sign-in screen has a server selector: Flowly (flowly.net) by default, "Add a server" with
  a URL (web app or API; the app's `config.json` reveals the API), several servers kept with separate sessions, switch
  between them from the menu. Only https, except localhost for development.
- **Desktop (Tauri)**: a list of servers in its config folder, each with API URL, app URL and its own session; the
  OAuth endpoints are discovered from the server's `/.well-known/oauth-authorization-server`; tray "Servers" submenu to
  switch, add, sign out, remove; existing single-server sessions migrate to the flowly.net entry.

---

## 24b. Browser extension (flowly-chrome)

Manifest V3, plain JavaScript, no build step. Options: Flowly address (API found like the mobile app), a personal
token (`flw_…`), the user's own OpenRouter key and model (default `google/gemini-2.5-flash`); host permission requested
only for that server (OpenRouter is declared). The popup reads the selection or the page's main text (≤ 24k chars)
when the user clicks, asks the model for `{"tasks": [{title, date, minutes, priority, notes}]}` (only the reader's own
actions; no past dates; ≤ 20), shows them for review (untick, edit title and date, pick workspace and project), then
creates them in one `tasks/bulk/` call assigned to the user, "high" mapped to the workspace's top priority, notes
ending with the source page's title and URL. Right-click a selection → "Send to Flowly" opens the same review in a
window. CI checks and zips it; `v*` tags publish the zip.

## 25. Guide and help

"Make Flowly yours" checklist (`/api/my/guide/`): plan your day, create projects, add tags, invite your team, connect
a calendar, set your home town, bring work in, meet your concierge, use Flowly from an AI assistant, track time; each
with why, a link and done state; Pro badge when plans are on.

---

## 26. Non-functional requirements

### 26.1 Security

- **Outbound HTTP** to user-supplied URLs goes through one helper that resolves the host and refuses private, loopback,
  link-local and metadata addresses (SSRF), turns non-2xx into a typed error with `retry` semantics, and sets a
  user agent.
- Secrets (integration credentials, SSO client secrets, user Stripe keys, calendar tokens, instance secrets) are
  encrypted at rest with a key derived from `DJANGO_SECRET_KEY` (Fernet) and never returned by the API (only
  `has_secret`).
- Webhooks in and out are signed and verified (GitHub, Slack, Stripe, Telegram secret token, Flowly's own HMAC).
- OAuth everywhere uses PKCE; OAuth/SSO state lives server-side and is single-use with a short TTL.
- Tokens handed to the browser via redirect go in the URL fragment, never the query string.
- Rate limits on public endpoints (invite codes, SSO). Non-members get 404 on workspace resources.
- No secrets in repositories: `.env` ignored, `.env.example` documented; CI must not need secrets to run tests.
- Chat outputs never ping (Discord `allowed_mentions`, Slack escaping); user text is escaped in Telegram HTML.

### 26.2 Testing

Django `APITestCase` suites per app, through the API wherever possible; external services mocked at the HTTP layer
(a fake `requests.request` recording calls) or at the model call (`chat`), including a fake OIDC provider with a real
RSA key. Tests must pass with plans off (the default) and declare `PLANS_ENABLED=True` where they test plan rules.
CI runs the suite on every push.

### 26.3 Internationalization

Dictionaries per area (`board`, `shell`, `workspace`, `admin`) for it/es/fr/de/pt keyed by the English string; the
`t` pipe interpolates `{name}` parameters; dates through a locale-aware pipe. Every new UI string is added in all five
languages in the same change.

### 26.4 Design system

CSS custom properties for colours (`--ink`, `--muted`, `--surface`, `--surface-2`, `--line`, `--accent`, `--brand`
(workspace colour), `--ok`, `--warn`, `--danger`, shadows) in light and dark, one `--page-width` (1100 px) for every
constrained page while My day and the board are full width, panels with headers (`.panel > h2`), rows, segmented
controls, chips, `.tip` hover labels for icon-only buttons (also aria-labels), the top bar of line icons with the
active item showing its label, toasts. Checkboxes and radios never inherit text-field padding.

### 26.5 Deployment and self-hosting

- `docker compose up -d` with `.env` (secret key, allowed hosts, CORS/CSRF origins, `PUBLIC_URL`, `API_PUBLIC_URL`,
  optional AI key and model, Telegram token, `PLANS_ENABLED`); migrations on start; the worker as a second service.
- Frontend container serves the build and writes `config.json` from `API_URL`.
- GitHub Actions: backend tests; Docker images to GHCR (`ghcr.io/flowly-org/flowly-backend`, `…/flowly-frontend`) on
  `main` and tags; Netlify deploy only when its secrets exist (forks skip it); mobile builds (signed APK when the
  keystore secrets exist); desktop installers for macOS, Windows and Linux attached to tagged releases.

---

## 27. Build order

1. **Foundations**: Django project, custom user, JWT auth, workspaces/memberships/invitations, instance settings,
   docker compose, CI with tests. Angular shell with auth pages, top bar, theme, i18n scaffolding, icon component.
2. **Tasks and the board**: projects, tag groups with system statuses/priorities, tasks with visibility, My day and the
   workspace board with swimlanes, drag and drop, backlog, carry over, task dialog, comments, attachments, sharing
   links, notifications.
3. **Planning tools**: bulk add syntax, templates, recurrence, checklists, estimates and day load, reminders,
   dependencies, search palette, timeline.
4. **Time and money**: timers, time page, billing, invoices, e-invoice formats, pay links.
5. **Integrations core**: integration model, delivery queue and worker, outgoing/incoming webhooks, Slack, Discord,
   email in, GitHub.
6. **Calendars**: providers, sync, strip, preparation, feed out, home and travel time, time blocking and auto-plan.
7. **Flo**: LLM client with providers and quotas, tool loop with budgets, Telegram, in-app panel, morning brief and
   card, notes/photo/voice extraction, plan-my-week, memory, appointment briefings, weekly review and insights.
8. **Focus**: ADHD mode, Now card, nudges, simple view and swipe deck.
9. **Teams**: team bots (Telegram group, Slack app), agile rituals, wiki, Donetick, Trello/Asana import, project move
   and promote.
10. **Platform**: MCP server and OAuth, delegation agent, SSO, admin pages with activation funnel, plans switch and
    Stripe, invite codes.
11. **Clients and release**: Capacitor apps and Tauri desktop with multiple servers, apps page, Docker images on GHCR,
    the static site.

At every step: tests for the new behaviour, all five translations for new strings, line icons only, and the feature
working on a phone.
