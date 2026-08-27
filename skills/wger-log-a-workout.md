---
name: wger-log-a-workout
description: Log a training session against a wger routine so the sets are attached to the plan they came from, not stranded as freestanding logs.
api: Wger REST API v2
base_url: https://wger.de/api/v2
generated: '2026-08-27'
method: generated
source: openapi/wger-openapi.yml
operations:
  - routine_list
  - routine_structure_retrieve
  - routine_date_sequence_display_list
  - workoutsession_create
  - workoutlog_create
  - workoutlog_list
  - workoutsession_list
auth: 'Authorization: Token <key>  (or Bearer <jwt>)'
---

# Log a workout in wger

## Why this skill exists

wger will happily accept a `workoutlog` that names only an exercise, a weight and
some reps. It will be saved, and it will be **invisible** — no routine view, no
routine statistic and no progression calculation can see a log that is not
attached to the plan it came from. Attaching it is three extra fields, and this
is the whole difficulty of logging in wger.

## Before you start

Check what instance you are talking to. wger is self-hosted and every deployment
is upgraded independently:

```
GET /api/v2/version/
GET /api/v2/min-server-version/
```

## Steps

### 1. Find the routine

```
GET /api/v2/routine/            # routine_list
```

Take the `id` of the routine the trainee is following. Routines have `start` and
`end` dates; the active one is the one whose range contains today.

### 2. Read what is prescribed today

```
GET /api/v2/routine/{id}/structure/              # routine_structure_retrieve
GET /api/v2/routine/{id}/date-sequence-display/  # routine_date_sequence_display_list
```

`structure` resolves the five-level tree (`Routine -> Day -> Slot -> SlotEntry ->
Config`) into what it actually prescribes. From it you need, per planned set:

- `slot_entry` id
- `exercise` id
- `iteration`
- the prescribed reps / weight / RiR (to send back as the `*_target` fields)

### 3. Open the session

```
POST /api/v2/workoutsession/    # workoutsession_create
{
  "routine": <routine id>,
  "day": <day id>,
  "date": "2026-08-27",
  "impression": "good",
  "notes": "..."
}
```

`impression` is `bad` | `neutral` | `good` — the trainee's own verdict, which no
aggregate over the logs can reconstruct, so ask for it rather than omitting it.
**One session per routine per date.** If a session already exists for today,
reuse its id instead of creating a second one:

```
GET /api/v2/workoutsession/?date=2026-08-27    # workoutsession_list
```

### 4. Log each set

```
POST /api/v2/workoutlog/        # workoutlog_create
{
  "session": <session id>,
  "routine": <routine id>,
  "slot_entry": <slot entry id>,
  "iteration": <iteration>,
  "exercise": <exercise id>,
  "repetitions": "8",
  "weight": "60.00",
  "weight_unit": <weight unit id>,
  "rir": "1"
}
```

Send `routine`, `slot_entry` and `iteration` **every time**. Without them the set
is freestanding, and nothing in the routine surface will ever show it.

Two units to get right:

- `weight_unit` — the weight is stored in the unit you name, with **no
  conversion**. Look ids up once at `GET /api/v2/setting-weightunit/`.
- `repetitions_unit` — says what `repetitions` counts. Without it a 60-second
  plank is stored as 60 repetitions. Look ids up at
  `GET /api/v2/setting-repetitionunit/`; since wger 2.5 those records carry
  `unit_type` and `multiplier` so you can tell a time unit from a count.

### 5. Read it back

```
GET /api/v2/workoutlog/?date=2026-08-27   # workoutlog_list
GET /api/v2/routine/{id}/logs/            # routine_logs_list
```

If the set shows in `workoutlog_list` but not in `routine_logs_list`, step 4
dropped the plan linkage. Fix it in place:

```
PATCH /api/v2/workoutlog/{id}/   # workoutlog_partial_update
{"routine": ..., "slot_entry": ..., "iteration": ...}
```

## Rules

- **Errors are not RFC 9457.** A `400` returns `{"<field>": ["message"]}`; a
  `401`/`403`/`404`/`429` returns `{"detail": "message"}`. Branch on the status
  and, for `400`, walk the field map. See `errors/wger-problem-types.yml`.
- **There is no idempotency key.** A retried `POST /api/v2/workoutlog/` creates a
  second set. On a timeout, `GET` the session's logs and reconcile before
  retrying — never blind-retry a create.
- **Nothing here is reversible.** There is no undo. `DELETE
  /api/v2/workoutsession/{id}/` deletes the session *and every set logged inside
  it*. Prefer `PATCH` to correct a value; escalate a delete to the human.
- **Rate limits do not apply here.** Only auth, registration and the ingredient
  endpoints are throttled. Everything in this skill is unthrottled.
