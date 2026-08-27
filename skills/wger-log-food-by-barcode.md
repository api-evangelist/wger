---
name: wger-log-food-by-barcode
description: Log food into a wger nutrition diary starting from a retail barcode, in the portion the user actually ate rather than in grams.
api: Wger REST API v2
base_url: https://wger.de/api/v2
generated: '2026-08-27'
method: generated
source: openapi/wger-openapi.yml
operations:
  - ingredient_list
  - ingredientinfo_list
  - ingredientinfo_retrieve
  - ingredientweightunit_list
  - nutritionplan_list
  - nutritionplan_create
  - nutritiondiary_create
  - nutritiondiary_list
  - nutritionplan_nutritional_values_retrieve
auth: 'Authorization: Token <key>  — the catalog reads are anonymous, the diary write is not'
---

# Log food by barcode in wger

## Why barcode first

Name search over a food database is unreliable in every language. wger imports
its ingredient catalog from Open Food Facts and keeps the retail barcode on the
record as `code`, so an exact lookup is available and is always the better
first move.

## Steps

### 1. Resolve the barcode

```
GET /api/v2/ingredientinfo/?code=5449000000996     # ingredientinfo_list
```

No credential needed — the ingredient catalog is anonymous. Use the **info**
projection: it embeds the weight units and the image, which you need in step 3.

Nothing back? The catalog is strong on branded packaged goods and thin on
supermarket private labels, and coverage varies a lot by country. Fall back to
a relevance-ranked name search:

```
GET /api/v2/ingredientinfo/?language__code=en&name__search=coca+cola
```

The catalog is **read-only over REST** — `/api/v2/ingredient/` answers
`GET, HEAD, OPTIONS` and nothing else. You cannot create a missing ingredient
through the API; a `POST` returns `405`. Missing items are added upstream at
Open Food Facts and flow in on the next sync.

Note the throttles on this step: 120 requests/min on the list endpoints, 300 on
detail. They are the only endpoints in this skill that are limited, and there is
no `RateLimit-*` header to read ahead — on a `429`, honour `Retry-After`.

### 2. Find or create the plan

```
GET /api/v2/nutritionplan/         # nutritionplan_list
```

If the user has none:

```
POST /api/v2/nutritionplan/        # nutritionplan_create
{
  "description": "Daily log",
  "only_logging": true,
  "goal_energy": 2400,
  "goal_protein": 160,
  "goal_fiber": 30
}
```

`only_logging: true` is the right default for someone who just wants to record
what they ate rather than plan meals in advance.

### 3. Decide grams or portions

```
GET /api/v2/ingredientweightunit/?ingredient=<id>   # ingredientweightunit_list
```

This returns the portions wger knows for that ingredient — slice, cup, can —
and the grams each one weighs. It is the only way to discover the ids, and it
matters: `amount` is grams **unless** you send `weight_unit`, in which case it
counts portions. Sending `amount: 2` with a `weight_unit` for "slice" logs two
slices; sending it without logs two grams.

### 4. Write the diary entry

```
POST /api/v2/nutritiondiary/       # nutritiondiary_create
{
  "plan": <plan id>,
  "ingredient": <ingredient id>,
  "meal": <meal id or null>,
  "weight_unit": <unit id or null>,
  "amount": "330.00",
  "datetime": "2026-08-27T07:30:00+02:00"
}
```

`datetime` takes a full timestamp with offset. Omit it and the server uses now;
send a bare date and it anchors at noon.

### 5. Read the day back

```
GET /api/v2/nutritiondiary/?plan=<id>                        # nutritiondiary_list
GET /api/v2/nutritionplan/{id}/nutritional_values/           # nutritionplan_nutritional_values_retrieve
```

Report the totals from the server rather than adding up your own — entries
logged in a portion unit have to be scaled by what that unit weighs, and the
server already does it.

## Rules

- **Correct in place, do not delete and re-add.** `PATCH
  /api/v2/nutritiondiary/{id}/` fixes an amount, a time, the ingredient or the
  meal. A `DELETE` cannot be undone — there is no restore anywhere in this API.
- **No idempotency key.** A retried `POST` logs the food twice. On a timeout,
  list the day's entries and reconcile before retrying.
- **Errors are the DRF envelope, not RFC 9457.** `400` returns a field map;
  everything else returns `{"detail": ...}`.
- **Barcodes are GTIN/EAN/UPC.** Send the digits exactly as printed, including
  leading zeros.
