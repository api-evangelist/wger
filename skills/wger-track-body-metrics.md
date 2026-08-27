---
name: wger-track-body-metrics
description: Record and read a wger user's body weight and tape measurements, including creating the measurement categories the entries hang off.
api: Wger REST API v2
base_url: https://wger.de/api/v2
generated: '2026-08-27'
method: generated
source: openapi/wger-openapi.yml
operations:
  - weightentry_create
  - weightentry_list
  - weightentry_partial_update
  - measurement_category_list
  - measurement_category_create
  - measurement_list
  - measurement_create
  - measurement_partial_update
  - userprofile_retrieve
  - userprofile_partial_update
auth: 'Authorization: Token <key>  (or Bearer <jwt>)'
---

# Track body weight and measurements in wger

## Two different shapes

Body weight is flat — a date and a number. Everything else measured with a tape
is two-level: the user owns the **categories** (Waist, Chest, Bicep), each with
its own unit, and entries hang off them. There are no built-in categories, so a
fresh account has nowhere to put a waist measurement until you create one.

## Body weight

```
GET  /api/v2/weightentry/?ordering=-date&limit=30    # weightentry_list
POST /api/v2/weightentry/                            # weightentry_create
{"date": "2026-08-27", "weight": "82.40"}
```

The entry has exactly two fields. Correct a mistake with:

```
PATCH /api/v2/weightentry/{id}/    # weightentry_partial_update
```

wger's own calorie logic reads the most recent weight entry alongside the
profile, so logging weight is what keeps the rest of the app honest.

## Measurements

### 1. Find or create the category

```
GET /api/v2/measurement-category/                # measurement_category_list

POST /api/v2/measurement-category/               # measurement_category_create
{"name": "Waist", "unit": "cm"}
```

`unit` is free text — whatever you write is what gets displayed. Pick one per
category and keep it; the API does not convert, and changing it later
reinterprets every existing entry under that category.

### 2. Log an entry

```
POST /api/v2/measurement/          # measurement_create
{
  "category": <category id>,
  "date": "2026-08-27",
  "value": "84.00",
  "notes": "morning, fasted"
}
```

### 3. Read a series

```
GET /api/v2/measurement/?category=<id>&ordering=-date     # measurement_list
```

Filters are AND-joined and take one value each. Watch the boolean rule if you
add any: wger accepts `True` and `False` **case-sensitively** and silently
ignores `1`, `0` and `false` — a bad boolean gives you a wrong result set, not
an error.

## Profile context

```
GET   /api/v2/userprofile/         # userprofile_retrieve
PATCH /api/v2/userprofile/         # userprofile_partial_update
```

Height, birthdate, sex, activity levels and the calorie target live here. Note
that `PUT` on this collection is `userprofile_update_legacy` and is the **only
deprecated operation in the entire wger API** — use `PATCH`.

## Rules

- **Correct, do not delete.** `PATCH` fixes a value, a date, a note, or (on a
  measurement) the `category` when something was filed in the wrong place.
  `DELETE /api/v2/measurement-category/{id}/` takes every entry under it and
  cannot be undone.
- **No idempotency key.** Two POSTs on the same date make two entries; wger does
  not deduplicate. Read the day back before retrying.
- **These endpoints are unthrottled.** Only auth, registration and the
  ingredient catalog are rate-limited.
