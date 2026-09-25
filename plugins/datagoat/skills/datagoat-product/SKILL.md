---
name: datagoat-product
description: Use when building or running a product that asks Datagoat about its own customers' cases on a schedule - score every open deal each morning, flag accounts likely to churn this month, rank machines likely to fault this week - and when keeping it healthy afterwards. Covers one namespace per customer, fitting once and scoring from the model with no fit, what a scheduled job must send, building each case's features from what was known at that moment, reporting outcomes, refitting with refit_of, reading dg_drift, and renewing or retiring models. For a single question on one table, use datagoat-ask.
---

# Ship and run a product on Datagoat

A product does the same six things for every customer: find the questions, keep the customer
apart, fit once, score from the model, explain, and close the loop. Guide:
https://datagoat.io/docs/build and https://datagoat.io/docs/run.

**Rule you must never break:** never state a chance, level, reason, range or direction the
response does not contain, and never let your own code round or reorder one. Your code owns the
thresholds and the actions; Datagoat owns the numbers.

## Ship it

1. **Find the questions.** `dg_suggest` on a customer's table lists yes/no questions; ask the
   `worth_asking` ones first ("worth trying", never "it can answer"). A non-empty `defined_by`
   means other columns restate the outcome: the answer would tell the customer nothing new.
   `dg_preflight` checks the table's grain and which columns are ambiguous before the first ask.
2. **One namespace per customer.** `namespace: "<customer id>"` (lowercase letters, digits, `-`,
   `_`; 1 to 64). A model fitted in one namespace never answers in another. Calls that take a
   `model_ref` find its namespace themselves.
3. **Fit once.** The first ask on a record fits a model; `model_ttl_days` (1 to 365, default 90)
   sets how long it answers. Keep the answer's `model_ref` and `model_expires_at`.
4. **Score from the model.** A question `{"type": "yesno", "model_ref": "mr1_…"}` fits nothing.
   Send new cases as `cases.rows` (no `data` needed), or send today's record with the fit's shape
   settings and name `cases.ids`. Answers list `model_columns`: the columns a case must carry. A
   case missing some is refused with every missing column named (`row_not_scoreable`).
   A test key (`dgk_test_…`) scores from the model only against a sample record (`cases.ids`);
   rows with no record are the caller's own data and need a live key.
5. **Explain.** Each case's `reasons` (with `range.text`) explain its chance; the answer's
   `pattern` and the case's `pattern_match` describe the at-risk group.

## Build each case from what was known then

A case scored today may only use what was known today. Otherwise held-out accuracy looks better
than it will be.

- Each feature comes from events at or before the scoring moment; the outcome comes strictly after.
- For an event log, the `snapshots` shape does this for you: it reads the log as of each moment you
  list, holds out whole cases, and says so in `quality.held_out.whole_cases_by`.
- For a table you built yourself with several rows per case (a deal at each stage), name the case
  with `group_column`, or held-out numbers overstate accuracy.
- Leave out any column written after the outcome is known (a close reason, a refund id, a
  cancellation date). If `dg_suggest` lists it in `defined_by`, it restates the outcome.

## Run it

- **Report outcomes** as you learn them: `dg_report_outcomes` with the `model_ref` and
  `{entity_id, outcome, observed_at}` rows; give each an `event_id` so a retry is safe.
- **Refit on a schedule** with the new record and `refit_of: "<old model_ref>"`, then read
  `dg_drift` for the new model:
  - `keep` → `dg_extend_model` on the model you keep, and carry on;
  - `refit` → switch to the new `model_ref`;
  - `abandon` → stop asking that question;
  - `no_check_yet` → nothing has been compared; refit first, and never renew on it.
- **Lifetime is a privacy limit, not a quality check.** Set it longer than the refit cycle.
  `dg_extend_model` renews before expiry; an expired model is not revived.
- **A customer leaves:** `dg_delete_model` removes the model and its outcome record now.

## Data it keeps

Inline rows are deleted when the call ends; a stored dataset 24 hours after its last use; a fitted
model holds cut points and category values, never rows. Keep your own copy of each customer's
record: the scheduled job sends it (or the day's new rows) each time.

## Cost

One decision per answered case; a fit the first time a record is asked about (1,000 free a month,
then $0.01). A `model_ref` question runs no fit. Refusals and `not_yet` bill no decisions.
