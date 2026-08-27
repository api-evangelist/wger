---
name: wger-build-a-routine
description: Create a wger training routine end to end — days, slots, exercises and the per-iteration progression configs that make weight and reps advance over time.
api: Wger REST API v2
base_url: https://wger.de/api/v2
generated: '2026-08-27'
method: generated
source: openapi/wger-openapi.yml
operations:
  - routine_create
  - day_create
  - slot_create
  - slot_entry_create
  - sets_config_create
  - repetitions_config_create
  - max_repetitions_config_create
  - weight_config_create
  - rir_config_create
  - routine_structure_retrieve
  - exerciseinfo_list
  - setting_weightunit_list
  - setting_repetitionunit_list
auth: 'Authorization: Token <key>  (or Bearer <jwt>)'
---

# Build a training routine in wger

## The shape you are building

```
Routine ──< Day ──< Slot ──< SlotEntry ──< Config (×10 collections)
```

Five levels to reach a prescribed number. A **Slot** is one position in a day
(share a slot across two entries and you have a superset). A **SlotEntry** binds
one exercise into that slot. Everything the entry prescribes — sets, reps,
weight, RiR, rest — is a separate per-iteration **Config** record.

## Steps

### 1. Create the routine

```
POST /api/v2/routine/           # routine_create
{
  "name": "Upper/Lower 4x",
  "description": "...",
  "start": "2026-09-01",
  "end": "2026-11-30",
  "fit_in_week": true,
  "is_template": false,
  "is_public": false
}
```

`is_template` marks it a reusable blueprint (it then appears in
`/api/v2/templates/`); `is_public` additionally offers it to every user of the
instance (`/api/v2/public-templates/`). Both default to false — leave them there
unless the user asked.

### 2. Add days

```
POST /api/v2/day/               # day_create
{
  "routine": <routine id>,
  "name": "Upper A",
  "order": 1,
  "is_rest": false,
  "need_logs_to_advance": false
}
```

`need_logs_to_advance` holds the plan on that day until sets are actually
logged, instead of advancing by the calendar. It is the right choice for someone
who trains irregularly, and the wrong one for a fixed weekly schedule — ask.

### 3. Add a slot, then bind an exercise to it

```
POST /api/v2/slot/              # slot_create
{"day": <day id>, "order": 1, "comment": ""}

POST /api/v2/slot-entry/        # slot_entry_create
{
  "slot": <slot id>,
  "exercise": <exercise id>,
  "order": 1,
  "type": "normal",
  "repetition_unit": <id>,
  "weight_unit": <id>,
  "repetition_rounding": "1.00",
  "weight_rounding": "2.50"
}
```

Find the exercise id first — and search the **info** projection, because
`Exercise` itself carries no name:

```
GET /api/v2/exerciseinfo/?language__code=en&name__search=bench+press   # exerciseinfo_list
```

`weight_rounding` snaps whatever a progression computes to a loadable step. Set
it to the smallest plate pair the trainee actually has (2.5 kg is the common
default); without it the plan will prescribe 62.37 kg.

### 4. Prescribe the numbers

Each quantity is its own collection, and each has a `max-` sibling for ranges:

```
POST /api/v2/sets-config/           {"slot_entry": <id>, "iteration": 1, "value": "3"}
POST /api/v2/repetitions-config/    {"slot_entry": <id>, "iteration": 1, "value": "8"}
POST /api/v2/max-repetitions-config/{"slot_entry": <id>, "iteration": 1, "value": "12"}
POST /api/v2/weight-config/         {"slot_entry": <id>, "iteration": 1, "value": "60.00"}
POST /api/v2/rir-config/            {"slot_entry": <id>, "iteration": 1, "value": "2"}
```

"3 sets of 8–12 at 60 kg, 2 in reserve" is five records. That is the model, not
a workaround.

### 5. Make it progress

A config record can carry `operation`, `step` and `repeat` so the value advances
by iteration, and `requirements` to gate that advance on what was actually
logged:

```
POST /api/v2/weight-config/
{
  "slot_entry": <id>,
  "iteration": 2,
  "value": "2.50",
  "operation": "+",
  "step": "abs",
  "repeat": true,
  "requirements": <gate>
}
```

`operation` is `+` | `-` | `r` (plus, minus, replace) and `step` is `na` | `abs`
| `percent` — read them from `OperationEnum` and `StepEnum` in the schema.
`requirements` is declared nullable with no fixed shape in the OpenAPI, so read
its accepted form from the running instance or from the routine API guide
(https://wger.readthedocs.io/en/latest/api/routines.html) rather than guessing;
the provider documents it as gating on any of `repetitions`, `weight`, `rir` and
`rest`.

With `requirements`, the step only fires on a workout that met the prescription —
which is how wger expresses autoregulation. Gated progressions advance exactly
one step per qualifying workout; they do not back-fill skipped iterations.

### 6. Verify the whole tree

```
GET /api/v2/routine/{id}/structure/   # routine_structure_retrieve
```

Read it back and check the resolved prescription against what you intended.
Do this before telling the user the routine is ready — the tree is deep enough
that a mis-set `order` or a config on the wrong `slot_entry` is easy to make and
invisible until someone trains.

## Rules

- **Order matters and is yours to manage.** `order` on days, slots and entries is
  not auto-assigned. Two records with the same `order` sort arbitrarily.
- **Deletes cascade and cannot be undone.** `DELETE /api/v2/routine/{id}/` takes
  every day, slot, entry and config with it; the same is true one level down for
  days and slots. There is no restore, no trash and no soft delete anywhere in
  this API. Confirm with the human before any delete.
- **No idempotency key.** A retried create makes a duplicate. If a POST times
  out, `GET` the parent's children and reconcile.
- **Check the instance version first.** `need_logs_to_advance`, gated
  progressions and the `unit_type`/`multiplier` fields on repetition units are
  recent; an older self-hosted instance will not have them. `GET /api/v2/version/`.
