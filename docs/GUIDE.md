# Setup & Reference Guide

This document walks you from zero to a running TickTick Autoscheduler workflow, then provides a reference for every node and rule it uses. The order matters — each section assumes the previous ones are done.

**Contents**

1. [Prerequisites](#1-prerequisites)
2. [API setup](#2-api-setup)
   - [2.1 Google Cloud Console](#21-google-cloud-console)
   - [2.2 TickTick Developer Center](#22-ticktick-developer-center)
   - [2.3 Obtain a TickTick access token](#23-obtain-a-ticktick-access-token)
3. [n8n credentials](#3-n8n-credentials)
4. [Import the workflow & assign credentials](#4-import-the-workflow--assign-credentials)
5. [TickTick tag setup](#5-ticktick-tag-setup)
6. [The `Defaults` Set node](#6-the-defaults-set-node)
7. [The Schedule Trigger](#7-the-schedule-trigger)
8. [Workflow walkthrough (node by node)](#8-workflow-walkthrough-node-by-node)
9. [Activating the workflow](#9-activating-the-workflow)
10. [Limitations & known quirks](#10-limitations--known-quirks)

---

## 1. Prerequisites

Before you start, make sure you have:

- A running n8n instance reachable over HTTPS (self-hosted or n8n Cloud). The OAuth flows below need your n8n's public URL — referred to throughout this document as `<your-n8n-domain>`, e.g. `n8n.example.com`.
- A **TickTick paid account.** The free tier's API behavior is undocumented and tends to choke on a typical project count; use a paid account for predictable results.
- A **Google account** with Calendar access. The workflow only supports Google Calendar — see [Section 6.2](#62-googlecalendarids) for how to bridge other calendars.
- Comfort with n8n basics: importing a workflow JSON, creating a credential, editing a Set node, and testing a workflow with the *Execute step* button.

---

## 2. API setup

You'll create one OAuth client on each side (Google + TickTick) so the workflow can read your calendar and read/write your TickTick tasks.

### 2.1 Google Cloud Console

#### 2.1.1 Create or select a project

1. Open <https://console.cloud.google.com/>.
2. Click the project picker at the top → **New Project** (or pick an existing project you want to attach this OAuth client to).
3. Give it a name like `n8n-calendar` and create.

#### 2.1.2 Enable the Calendar API

1. Left menu → **APIs & Services** → **Library**.
2. Search for **Google Calendar API** → click → **Enable**.

#### 2.1.3 Configure the OAuth consent screen

1. Left menu → **APIs & Services** → **OAuth consent screen** (called *Audience* in newer Cloud Console versions).
2. **User type:**
   - **Internal** if your Google account is part of a Google Workspace organization and you only need to authenticate that account. No verification or test-user list required.
   - **External** otherwise. While in *Testing* status, add your Google account under **Test users** — without this the consent flow rejects you.
3. **Scopes:** add `https://www.googleapis.com/auth/calendar.readonly`. The workflow only reads free/busy data, never writes calendar events.

#### 2.1.4 Create the OAuth Client ID

1. Left menu → **APIs & Services** → **Credentials** → **Create credentials** → **OAuth Client ID**.
2. **Application type:** Web application.
3. **Name:** anything (e.g. `n8n-calendar`).
4. **Authorized redirect URIs:** add exactly:

   ```
   https://<your-n8n-domain>/rest/oauth2-credential/callback
   ```

   This is n8n's universal OAuth2 callback path; the same URL works for every OAuth2 credential you'll ever create on this n8n instance.
5. Click **Create**. Copy the **Client ID** and **Client Secret** that appear in the dialog. Save them somewhere safe — you'll need them in [Section 3.1](#31-google-calendar-oauth2-api).

> Google's redirect URI changes can take up to 5 minutes to propagate. If your first OAuth attempt below gets `redirect_uri_mismatch`, wait a few minutes and retry.

### 2.2 TickTick Developer Center

> **Reference:** TickTick's own OAuth2 documentation lives at <https://developer.ticktick.com/api#/openapi?id=authorization>. The steps below are sufficient to get this workflow running, but if anything below diverges from current TickTick behavior, treat the upstream docs as authoritative.

#### 2.2.1 Register an app

1. Open <https://developer.ticktick.com/manage>.
2. Sign in with your TickTick account.
3. **New App.**
4. Fill in:
   - **App Name:** anything (e.g. `n8n autoscheduler`).
   - **Description:** optional.
   - **App Service URL:** `https://<your-n8n-domain>/`
   - **OAuth Redirect URL:** `https://<your-n8n-domain>/rest/oauth2-credential/callback`
5. Save. The app's **Client ID** and **Client Secret** are now visible — copy them somewhere safe.

### 2.3 Obtain a TickTick access token

TickTick's API uses Bearer-token authentication. n8n's HTTP Bearer Auth credential needs a static token, so you'll run TickTick's OAuth2 authorization-code flow once manually to obtain one. The token is valid for **180 days**.

#### 2.3.1 Get the authorization code

Build this URL (replace placeholders, all on one line):

```
https://ticktick.com/oauth/authorize
  ?client_id=<TICKTICK_CLIENT_ID>
  &scope=tasks:read tasks:write
  &state=anything
  &redirect_uri=https://<your-n8n-domain>/rest/oauth2-credential/callback
  &response_type=code
```

Open it in a browser. Approve the permissions. TickTick redirects to your n8n's callback URL with a `?code=...&state=...` query string. n8n will show an error page (because no in-flight OAuth dance exists for this URL on n8n's side) — that's fine. **Copy the `code` value from the address bar.** It expires in a few minutes, so move on quickly.

#### 2.3.2 Exchange the code for an access token

```bash
curl -X POST 'https://ticktick.com/oauth/token' \
  --user '<TICKTICK_CLIENT_ID>:<TICKTICK_CLIENT_SECRET>' \
  -d 'code=<CODE_FROM_PREVIOUS_STEP>' \
  -d 'grant_type=authorization_code' \
  -d 'redirect_uri=https://<your-n8n-domain>/rest/oauth2-credential/callback' \
  -d 'scope=tasks:read tasks:write'
```

The JSON response contains `access_token` (a UUID-shaped string). **This is your Bearer token.** Save it next to the Client ID/Secret.

Sanity-check it:

```bash
curl -H 'Authorization: Bearer <ACCESS_TOKEN>' https://api.ticktick.com/open/v1/project
```

You should get a JSON array of your TickTick projects. If you get `401 Unauthorized`, the token is wrong or the OAuth flow didn't complete; redo from 2.3.1.

---

## 3. n8n credentials

Now you have:
- Google: Client ID + Client Secret
- TickTick: Bearer access token (Client ID + Secret are kept for the next 180-day rotation)

Create two credentials inside n8n.

### 3.1 Google Calendar OAuth2 API

1. In n8n, open the **Credentials** tab in your workspace (top tab next to *Workflows* in current versions; left sidebar in older releases) → **Create Credential**.
2. Search **Google Calendar OAuth2 API** → select.
3. Paste **Client ID** and **Client Secret** from [Section 2.1.4](#214-create-the-oauth-client-id).
4. Click **Sign in with Google** → consent flow → grant Calendar read access.
5. Save. Name it something memorable like `Google Calendar`.

If the OAuth consent screen says "App not verified" and you have an Internal-type client, double-check the user type. External + Testing won't show this warning to test users either, but External + Production unverified will.

### 3.2 HTTP Bearer Auth (for TickTick)

1. In n8n's **Credentials** tab → **Create Credential**.
2. Search **HTTP Bearer Auth** → select.
3. **Bearer Token:** paste the TickTick access token from [Section 2.3.2](#232-exchange-the-code-for-an-access-token).
4. Save. Name it something like `TickTick API`.

---

## 4. Import the workflow & assign credentials

1. Download [`workflow.json`](../workflow.json) from this repo.
2. n8n → **Workflows** → top-right menu (`…`) → **Import from File** → select `workflow.json`.
3. The imported workflow is named **TickTick Autoscheduler** and is **inactive** by default. Don't activate it yet.
4. Open the workflow. Four HTTP nodes will display a red "credentials missing" badge. Click each in turn and assign the right credential:

   | Node | Credential type | Pick |
   |---|---|---|
   | `Query Ticktick projects` | HTTP Bearer Auth | TickTick API |
   | `Query Ticktick tasks` | HTTP Bearer Auth | TickTick API |
   | `Write back ticktick tasks` | HTTP Bearer Auth | TickTick API |
   | `Get Busy slots` | Google Calendar OAuth2 API | Google Calendar |

5. Save the workflow.

---

## 5. TickTick tag setup

The workflow reads two pieces of information from each TickTick task's tag list:

- whether the task should be moved at all (`dfix` says "leave me alone"), and
- how long the task takes when placed into a free slot (`d<n>` overrides the default).

Set up your tags in TickTick **before** you tweak the `Defaults` Set node, so you can think about defaults with the tag conventions already in mind.

### 5.1 Tag conventions

| Tag | Meaning |
|---|---|
| `dfix` | Pin the task. The scheduler will not rewrite its `startDate` / `dueDate`. |
| `d<n>` (e.g. `d05`, `d10`, `d15`, `d30`, `d45`, `d60`) | Sets the task's duration in minutes when the workflow places it into a free slot. Overrides the default. |
| *no `d<n>` tag* | Falls back to `DEFAULT_TASK_DURATION_MIN` from the Defaults Set node (configured in the next section). |

`dfix` has two distinct uses, depending on whether the task is **all-day** or **timed**:

- **All-day `dfix` task** — pinned to its day, but with no specific time. Use this for "I have to do this today, but it doesn't matter when". The task does *not* block other work — the scheduler can still place other tasks anywhere in your work hours that day. Typical fits: small checklist items, short follow-ups, errand reminders.
- **Timed `dfix` task** (with explicit start/end times) — pinned to its existing slot, and that slot is treated as a busy block. The scheduler will not place other tasks on top of it. Typical fits: doctor's appointment, scheduled call, fixed meeting.

You can mix tags freely. Other tags on a task (project labels, contexts like `urgent`, `errand`, etc.) are passed through untouched — the workflow only looks at `dfix` and `d<digits>`.

### 5.2 Recommended tag organization in TickTick

You'll typically end up with a `dfix` plus a handful of duration tags. Keeping them organized under a single parent tag in TickTick keeps the sidebar clean:

1. In TickTick's **Tags** sidebar, create a new parent tag — e.g. `duration`.
2. Create the children: `dfix`, `d05`, `d10`, `d15`, `d20`, `d30`, `d45`, `d60`.
3. Drag each child onto the `duration` parent so they nest underneath.

Sidebar result:

![TickTick tag tree organized under a `duration` parent](ticktick_tags.png)

(The numbers next to each tag in the screenshot are TickTick's own usage counters — they show how many open tasks currently carry each tag, and have no bearing on the workflow.)

The `duration` parent has no functional role for the workflow either — only the leaf tags matter. The parent is purely a TickTick organizational tool.

You don't have to predefine every duration value. The workflow accepts any `d<digits>` (e.g. `d12` for 12 minutes) — but having a few standard buckets makes tagging fast and consistent.

---

## 6. The `Defaults` Set node

This is where you configure the workflow's behavior. Open the **Defaults** node and edit the `jsonOutput` field. The default looks like:

```json
{
  "workSlots": {
    "monday":    [["09:00", "12:00"], ["13:00", "17:00"]],
    "tuesday":   [["09:00", "12:00"], ["13:00", "17:00"]],
    "wednesday": [["09:00", "12:00"], ["13:00", "17:00"]],
    "thursday":  [["09:00", "12:00"], ["13:00", "17:00"]],
    "friday":    [["09:00", "12:00"], ["13:00", "17:00"]],
    "saturday":  [],
    "sunday":    []
  },
  "googleCalendarIds": ["primary"],
  "PLAN_DAYS_AHEAD": 3,
  "DEFAULT_TASK_DURATION_MIN": 30,
  "TASK_BUFFER_MIN": 10,
  "TIMEZONE": "Europe/Budapest"
}
```

### 6.1 `workSlots`

Per-weekday array of `[start, end]` pairs in 24-hour `HH:MM` format. Each pair defines a window in which the scheduler is allowed to place tasks. Empty array = day off.

Examples:
- `"monday": [["09:00", "12:00"], ["13:00", "17:00"]]` — split for a lunch break.
- `"thursday": [["14:00", "18:00"]]` — a half day starting in the afternoon.
- `"saturday": []` — never schedule on Saturdays, even if a task is overdue.

Times are interpreted in the timezone you set in `TIMEZONE` (DST-aware via Luxon).

### 6.2 `googleCalendarIds`

Array of Google Calendar IDs whose busy time should block the scheduler. There are three forms a Google Calendar ID can take:

- **`"primary"`** — the main calendar of the Google account you authenticated with. Convenient: stays correct if you ever swap the auth account.
- **email address** (e.g. `"someone@gmail.com"`) — a personal calendar belonging to a different Google account. The auth account must have at least *See free/busy* permission on it (granted by the calendar owner).
- **`"c_<random>@group.calendar.google.com"`** — a secondary calendar in your own Google account, or a calendar you've subscribed to. Look the ID up in Google Calendar → calendar settings → *Integrate calendar* → *Calendar ID*.

#### Multiple Google accounts

To aggregate calendars across several Google accounts you own:

1. Sign in to the account that owns the calendar you want to share.
2. In Google Calendar settings for that calendar, share it with the account n8n authenticates as, granting at least *See free/busy*.
3. Add the calendar ID to `googleCalendarIds`.

The free/busy results from all listed calendars are unioned by the workflow.

#### Bridging non-Google calendars (Bitrix24, iCloud, Outlook, CalDAV, …)

The workflow speaks Google Calendar API only. But if your other calendar system publishes an iCal/ICS feed:

1. In Google Calendar, click *Other calendars → + → From URL* and paste the iCal feed URL.
2. Google maintains a subscribed copy and assigns it an internal ID of the form `c_<random>@group.calendar.google.com`.
3. Add that ID to `googleCalendarIds`.

The external calendar's busy times then flow in like a native Google calendar. **Caveat:** Google polls iCal feeds on its own schedule (typically several hours), so changes on the source side don't propagate instantly. For a real-time sync you'd need a different bridge.

There's also a more subtle quirk: the Google free/busy API only returns events marked **Busy**, not events marked **Free** (TRANSPARENT in iCal terms). Many iCal feeds default to TRANSPARENT, in which case those events are visible in your Google Calendar UI but invisible to this workflow. See [Section 10](#10-limitations--known-quirks).

### 6.3 `PLAN_DAYS_AHEAD`

How many days ahead, including today, the scheduler considers. With `PLAN_DAYS_AHEAD: 3` and today = Wednesday, the planning window ends Friday 23:59 in `TIMEZONE`. Tasks due *beyond* that date are ignored on this run.

A short window (1–3 days) gives a tight rolling plan — only the next few days get rewritten on each run, your further-future tasks stay untouched. A longer window (7+) pulls in more candidates but means each run can shuffle more of your future. Don't pick a window so long that the scheduler will rewrite tasks you're months away from doing.

### 6.4 `DEFAULT_TASK_DURATION_MIN`

Default duration in minutes used when a task has no `d<n>` tag. 30 is a reasonable starting point for a varied task list.

### 6.5 `TASK_BUFFER_MIN`

Free minutes inserted after each scheduled task before the next one can start. Acts as transition / context-switch time.

### 6.6 `TIMEZONE`

IANA timezone name (e.g. `"Europe/Budapest"`, `"America/Los_Angeles"`). Used for both work-slot interpretation and the Google free/busy query, and for formatting `startDate`/`dueDate` written back to TickTick. Independent of the n8n container's system timezone — the workflow uses Luxon to handle this regardless of where n8n is running.

---

## 7. The Schedule Trigger

Open the **Schedule Trigger** node. The default cron expression is:

```
0,30 8-18 * * 1-5
```

Meaning: every 30 minutes, between 08:00 and 18:00, weekdays only — interpreted in the n8n container's timezone (n8n's cron isn't `TIMEZONE`-aware out of the box for these custom expressions).

Adjust to taste. Common variants:

- `*/15 * * * *` — every 15 minutes, all day, every day. Overkill for most.
- `0 */2 8-18 * 1-5` — top of every other hour during business hours, weekdays.
- `0 9 * * 1-5` — once a day at 09:00 weekdays. Lightweight, good if you mostly trigger manually.

Higher frequency means the rolling plan reacts to calendar changes faster, but uses more API calls toward your TickTick rate limit and creates more noise in the n8n execution log.

---

## 8. Workflow walkthrough (node by node)

The workflow has 15 nodes. Below they're listed in execution order, with what each one does and the key rules that govern it.

### 8.1 `Schedule Trigger` and `When clicking 'Execute workflow'`

Two trigger nodes. The first fires on the cron expression you set in [Section 7](#7-the-schedule-trigger). The second is a manual trigger you press while testing — both lead into the same `Defaults` node downstream.

You can leave both in place. The manual trigger is useful for ad-hoc runs (e.g. after rearranging your day) without waiting for the next cron tick.

### 8.2 `Defaults`

A Set node with `mode: raw` outputting a single JSON object — the configuration described in [Section 6](#6-the-defaults-set-node). Every downstream node reads from this output by name (`$('Defaults').item.json.PLAN_DAYS_AHEAD` etc.), which means you only edit your config in one place.

### 8.3 `Get current Date & Time`

Returns the current moment, formatted in `TIMEZONE` (it reads `{{ $json.TIMEZONE }}` from `Defaults`). Currently the value isn't read by the `Schedule` Code node — it's exposed for future use or manual injection (e.g. running the workflow against a synthetic "now" for testing).

### 8.4 `Query Ticktick projects`

`GET https://api.ticktick.com/open/v1/project`

Returns the full list of your TickTick projects (TASK lists, NOTE lists, and any other kinds). Authenticated with the TickTick Bearer token. `retryOnFail` is enabled to ride out transient network blips.

### 8.5 `Filter projects`

An IF node that keeps only projects with `kind === 'TASK'`. Note-only projects (TickTick supports project-of-notes as a kind) are dropped here — there's nothing to schedule in them.

### 8.6 `Loop Over Items`

A `splitInBatches` node that iterates one project at a time through the next two nodes. Necessary because `Query Ticktick tasks` is parameterized by `projectId` and has to be called once per project.

### 8.7 `Query Ticktick tasks`

`GET https://api.ticktick.com/open/v1/project/{{ $json.id }}/data`

For each project, fetches its tasks (and any subtasks). Returns a JSON object with `project: {…}` and `tasks: [{…}]`. Bearer-authenticated. `retryOnFail` on, with `waitBetweenTries: 1000`.

### 8.8 `Flatten`

A Code node that takes the per-project responses and produces a flat array of tasks, with each task carrying its parent-project metadata.

Rules applied here:

- **Open tasks only:** parents and subtasks with `status === 2` (TickTick's "completed") are dropped.
- **Must have `dueDate`:** tasks (parent or subtask) without a `dueDate` are dropped — the scheduler can't reason about an undated task. *This is stricter than the historical "must have either startDate or dueDate" rule and reflects how TickTick treats fully-dated vs. all-day vs. floating items.*
- **Subtask flattening:** open subtasks of an open parent are emitted as independent items in the output, inheriting `projectId`/`projectName`/`parentTitle` etc. but with their own `taskId`, `title`, dates, priority. After this node a subtask is indistinguishable from a top-level task.
- **Date format normalization:** `startDate` and `dueDate` are reformatted to `yyyy-MM-ddTHH:mm:ss±zzzz` (no milliseconds, four-digit offset). The TickTick v1 API rejects the `.000` milliseconds that JavaScript's default ISO output produces.

### 8.9 `Get Busy slots`

`POST https://www.googleapis.com/calendar/v3/freeBusy`

Asks Google for the busy intervals across all calendars listed in `Defaults.googleCalendarIds`. Body is templated:

```jsonc
{
  "timeMin": "<now>",
  "timeMax": "<midnight + PLAN_DAYS_AHEAD days>",
  "timeZone": "<Defaults.TIMEZONE>",
  "items": [{ "id": "<calendar-id-1>" }, { "id": "<calendar-id-2>" }, …]
}
```

Authenticated with the Google Calendar OAuth2 credential. `retryOnFail` on.

The response contains a `calendars` map keyed by calendar ID, each with a `busy` array. Per-calendar errors (e.g. permission denied for a specific shared calendar) appear inline rather than as a top-level failure, so a typo'd ID doesn't break the whole run.

**TRANSPARENT events:** the free/busy endpoint only reports events marked **Busy**. Events marked **Free** are not visible. Most iCal subscriptions (Bitrix24, iCloud feeds, etc.) default to Free — see [Section 10](#10-limitations--known-quirks).

### 8.10 `Merge`

Combines four input streams into a single ordered list that flows into `Schedule`:

| Position | Source | Purpose |
|---|---|---|
| 0 | Defaults | Config (`workSlots`, `TIMEZONE`, durations…) |
| 1 | Get current Date & Time | Reserved (currently unused by Schedule) |
| 2 | Get Busy slots | Calendar busy intervals |
| 3+ | Flatten | The flat task list |

The `Schedule` node treats `items[0]` as config and everything after as data, distinguishing the busy-slot item by its `kind === 'calendar#freeBusy'` field.

### 8.11 `Schedule`

The brain. A Code node that runs the placement algorithm. Roughly:

1. Read config from `items[0]`.
2. Build "now", planning-window-end, and the work slots — all in `TIMEZONE` via Luxon, so DST and any container-vs-user timezone mismatch is handled correctly.
3. **Filter tasks** to a candidate set:
   - has a `dueDate` (the anchor for "when is this thing due")
   - is *not* tagged `dfix`
   - `dueDate ≤ windowEnd`

   `startDate` is *not* required at this stage — it's rewritten when the task is placed, so its presence on the input doesn't matter.
4. **Sort the candidates** into three groups: overdue → today → future. Overdue and today are ordered by priority descending; future is ordered by `dueDate` ascending.
5. **Compute free slots** by subtracting busy periods (Google free/busy ∪ active `dfix` task intervals) from the work slots.
6. **Place each task greedy first-fit** into the earliest free slot whose remaining length ≥ `duration + TASK_BUFFER_MIN`. After placement, the slot's start is advanced past the task plus the buffer.
7. **Leftovers** that don't fit anywhere become all-day items on the **last day** of the planning window, so they remain visible and get a fresh attempt next run.

Tasks emitted by this node have new `startDate`, `dueDate`, `done` (true if placed in a real slot, false if leftover), and `allDay` (true for leftovers). The original `dueDate` is *not* preserved anywhere — see [Section 10](#10-limitations--known-quirks).

#### Duration determination

For each candidate task:

- If a `d<digits>` tag is present (e.g. `d45`), the digits become the duration in minutes.
- Otherwise `DEFAULT_TASK_DURATION_MIN` is used.

#### `dfix` tasks: pinned, sometimes blocking

Tasks with the `dfix` tag aren't placed by the scheduler (they're filtered out in step 3). What happens to their existing interval depends on whether the task is timed or all-day:

- **Timed `dfix`** (`isAllDay === false`, with explicit start/end) — the existing interval is added to `busyPeriods`, so other tasks won't be scheduled on top of it. This is how a `dfix` doctor's appointment correctly blocks the scheduler.
- **All-day `dfix`** — the task stays pinned to its day but is *not* treated as busy. Other tasks can still be placed in your work hours that day. Use this for "must do today, no specific time" items: small checklist work, follow-ups, errand reminders.

### 8.12 `Loop Over Items1`

A `splitInBatches` after `Schedule`. Each batch contains one scheduled task that flows into the write-back chain.

### 8.13 `Write back ticktick tasks`

`POST https://api.ticktick.com/open/v1/task/{{ $json.taskId }}`

Updates the task in TickTick with the new `startDate`/`dueDate`/`isAllDay`. Body:

```jsonc
{
  "id": "<taskId>",
  "projectId": "<projectId>",
  "title": "<title>",
  "startDate": "<yyyy-MM-ddTHH:mm:ss±zzzz>",
  "dueDate":   "<yyyy-MM-ddTHH:mm:ss±zzzz>",
  "content":   "<original content>",
  "isAllDay":  "<true|false>"
}
```

Bearer-authenticated. `retryOnFail` on.

### 8.14 `Wait`

A 3-second pause between successive write-backs. This is the rate-limit cushion against TickTick's 30-req/min cap. With the wait in place, the workflow tops out at ~20 writes/min, comfortably under the limit. Loop Over Items1 → Wait → loop again is the pattern.

If you have many overdue tasks the first time you activate the workflow, this wait can make the initial run feel slow (a 60-task batch takes ~3 minutes). Subsequent runs only touch tasks that moved within the planning window, which is usually a handful.

---

## 9. Activating the workflow

Before flipping the switch, do a manual dry-run.

### 9.1 Dry-run with `Execute step`

1. In the workflow editor, click the **Schedule** Code node.
2. Click **Execute step** (or **Test step**, depending on n8n version). This runs the chain *up to and including* `Schedule`, but stops before `Write back ticktick tasks`. No TickTick data is modified.
3. Inspect the node's output. Each item is one task with proposed new `startDate`/`dueDate`. Check that:
   - `dfix` tasks aren't in the output.
   - The proposed times sit inside your `workSlots`.
   - The proposed times don't overlap your Google Calendar busy events.
   - The number of placed tasks roughly matches your expectation given how many of your tasks are non-`dfix` and within the window.

### 9.2 Full run

If the dry-run looks right:

1. Click **Execute workflow** at the bottom of the editor (or pick "Execute workflow from Schedule Trigger").
2. Watch the chain run end-to-end. Each task should produce a 200-OK response from the `Write back ticktick tasks` node.
3. Open your TickTick app and confirm the new times.

### 9.3 Activate the cron

Toggle the workflow to **Active** in the top-right. From now on the cron expression in `Schedule Trigger` drives execution.

---

---

## 10. Limitations & known quirks

- **Each run overwrites `startDate` / `dueDate` of every successfully placed task.** The original due date is not preserved anywhere — once a task gets pushed forward, you can't tell from TickTick that it used to be due last Tuesday. If you need to keep the original date, copy it into the task's content/description before relying on the workflow, or tag the task `dfix` so it never gets rewritten.
- After a task has been rescheduled, on the next run it's no longer "overdue" (its dueDate is now in the future). This is what makes the rolling-plan model work, but it also means missed-deadline accounting from inside TickTick is impossible.
- **Free/busy returns only Busy events.** Events marked "Free" (TRANSPARENT in iCal) are invisible to the workflow even though they show up on your Google Calendar UI. Most iCal subscriptions (Bitrix24, iCloud, Outlook feeds) default to Free — those events don't block scheduling. Workarounds: if you control the source side, mark the events as Busy; if you don't, switch to a different bridge.
- **iCal subscription staleness.** Google polls iCal feeds on its own schedule (often several hours). Last-minute changes on the source calendar may not be visible to the workflow yet.
- **Free/busy is not delegation-aware.** You only see calendars the OAuth-authorized account has explicit access to. Shared calendars need to be shared *to* that account (with at least *See free/busy* permission); domain-wide delegation isn't used here.
- **TickTick API rate limits.** Paid tier: 30 req/min. The workflow includes a 3-second `Wait` between writes which keeps it comfortably under that, but is not bulletproof — if other tools (mobile app, etc.) hit the API simultaneously you can still see 429s. The `retryOnFail` flags on the HTTP nodes catch most transient issues.
- **TickTick token rotation.** Bearer tokens expire after 180 days. When the workflow starts returning `401 Unauthorized` from TickTick, repeat [Section 2.3](#23-obtain-a-ticktick-access-token) to mint a fresh token, then update the n8n credential.
- **Holidays.** `workSlots` is a weekly recurring template; the workflow doesn't know about public holidays. If your `googleCalendarIds` includes a calendar that has the holiday as a Busy event, that's how the scheduler will see it — otherwise it'll happily schedule on May 1.
- **Time zones inside the n8n cron.** The Schedule Trigger's cron is interpreted in the n8n container's system timezone, which may not be the same as your `TIMEZONE` config. The *placement* logic is timezone-correct (Luxon-based), but the *trigger times* depend on the container. If you self-host and want both aligned, set `TZ=<your-IANA-zone>` on the n8n container.
