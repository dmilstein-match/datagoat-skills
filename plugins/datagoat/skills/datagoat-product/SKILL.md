---
name: datagoat-product
description: '"Score my customers nightly", "flag the accounts likely to churn each week", "one model per customer", "is the model still any good" - use when building or running a product that asks Datagoat about its own customers'' cases on a schedule - score every open deal each morning, rank machines likely to fault this week - and when keeping it healthy afterwards. Covers one namespace per customer, fitting once and scoring from the model with no fit, what a scheduled job must send, building each case''s features from what was known at that moment, reporting outcomes, refitting with refit_of, reading dg_drift, and renewing or retiring models. For a single question on one table, datagoat-ask.'
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
   `dg_map` on the table alone reports its grain and which columns are ambiguous before the first
   ask (`dg_preflight` does too, but it is deprecated and removed at contract 2.0.0).
2. **One namespace per customer.** `namespace: "<customer id>"` (lowercase letters, digits, `-`,
   `_`; 1 to 64). A model fitted in one namespace never answers in another. Calls that take a
   `model_ref` find its namespace themselves.
3. **Fit once.** The first ask on a record fits a model; `model_ttl_days` (1 to 365, default 90)
   sets how long it answers. Keep the answer's `model_ref` and `model_expires_at`.
3b. **Give each customer a profile (optional).** `dg_profile` with the namespace: `words` (what a
   case is called, the outcome in plain words, a phrase per column), `exclude` (columns no fit
   there uses), `fixed` (columns an action never changes, such as tenure: still read, no lever
   offered) and `display` (`chance`, or `bands` for end users, which also turns bands on). Answers
   there then carry `profile {hash, display}` and a `says` sentence per answered case ("customer n4 —
   chance it cancels: 50% (likely)"), quoted from the engine's values; a declined case has none.
4. **Score from the model.** A question `{"type": "yesno", "model_ref": "mr1_…"}` fits nothing.
   Send new cases as `cases.rows` (no `data` needed), or send today's record with the fit's shape
   settings and name `cases.ids`. Answers list `model_columns`: the columns a case must carry. A
   case missing some, or holding a value the model never saw, comes back `not_scoreable` naming them.
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
- **Or map it:** `dg_map` on a customer's logs (and their accounts table) proposes the reading,
  asks one question per ambiguous slot, and on answers builds a record read every
  `snapshot_every`, whose outcome is known only once its horizon has passed, plus a ready-to-run
  `ask` on today's open cases. The questions are the customer's to answer: present each with its
  `options`, never pick one in the product's code or the agent. Sources given as `fetch_url` are
  fetched again on each replay by `mapping_id` (with `closed_since`, the replay also returns the
  outcomes that became known, ready for `dg_report_outcomes`). Guide: https://datagoat.io/docs/map.
- For a table you built yourself with several rows per case (a deal at each stage), name the case
  with `group_column`, or held-out numbers overstate accuracy.
- Leave out any column written after the outcome is known (a close reason, a refund id, a
  cancellation date). If `dg_suggest` lists it in `defined_by`, it restates the outcome.

## Run it

- **Let a schedule run the loop.** `dg_schedule` with a confirmed `mapping_id` whose sources are
  all `fetch_url`, a `cadence` (`daily`, `weekly` or `monthly`) and `subject_kind`: each run
  replays the mapping (sources fetched again, `closed_since` the last run), fits (`refit_of` from
  the second run; skipped, billing no fit, when the labelled readings are unchanged), reads
  `dg_drift`, scores today's open cases and reports the outcomes that closed. The policy is fixed
  (keep renews `keep_model_ref` to the day after the next run; refit switches; abandon stops); do
  not re-implement it around a schedule. It needs a live API key made with "Can report outcomes"
  (`schedule_key_required` for an OAuth sign-in or a key without it); a mapping over uploads is
  `schedule_sources_not_refetchable`: say a schedule needs `fetch_url` sources rather than
  re-uploading on a timer. Read a schedule's state and last run in `dg_describe`'s `schedules[]`;
  `halted` carries `halt_code` (a billing code, `credential_revoked`, `source_unreachable`) for the
  user to fix, then schedule again. `dg_delete_schedule` ends it and deletes its stored key id and
  headers. Report its counts as the run gives them. By hand, the same calls follow.
- **Report outcomes** as you learn them: `dg_report_outcomes` with the `model_ref` and
  `{entity_id, outcome, observed_at}` rows; give each an `event_id` so a retry is safe. A batch
  is written whole or not at all: on `invalid_outcomes`, fix the rows its `errors` name (by
  `index` and `field`) and resend the batch, or send `partial: true` to write the valid rows now
  and read each row's `results`.
- **A model_ref stops answering** with `model_deleted` or `model_expired` (410): ask again with
  the record to fit a new model, and switch the job to the new `model_ref`.
- **Refit on a schedule** with the new record and `refit_of: "<old model_ref>"`, then read
  `dg_drift` for the new model:
  - `keep` → `dg_extend_model` on `keep_model_ref`, and carry on (a refit that measured worse,
    `refit_quality: "worse"`, `note_code: "refit_worse"`, is a keep: stay on the earlier model);
  - `refit` → switch to the new `model_ref` (`keep_model_ref` names it; `prior_unusable` means
    the earlier model expired or was deleted, so nothing was compared);
  - `abandon` → stop asking that question;
  - `no_check_yet` → nothing has been compared; refit first, and never renew on it.
- **Lifetime is a privacy limit, not a quality check.** Set it longer than the refit cycle.
  `dg_extend_model` renews before expiry; an expired model is not revived.
- **A customer leaves:** `dg_delete_model` removes the model now and closes its outcome record; the
  outcome reports already made stay in the engine's append-only ledger.

## Watching fits and models

- A scheduled job's first fit on a customer's record can run for minutes. Its pending answer
  carries `watch_url` (a page with the stages as the engine reports them, then the answer, and no
  rows) and its stages stream from `GET /v1/tasks/{task_id}/events`. Where the job or the host
  has its own task list, the reported stages mirror into it one to one, each labelled with the
  event's `message` word for word; a stage not yet reported is never shown or described, and
  nothing about a finding appears before the answer.
- On a long refit, the `watch_url` is the link to hand to whoever is waiting on it.
- `dg_track_record` and `dg_drift` answers carry a `watch_url` too: a page with the model's
  calibration (each band's observed rate against its mean chance, with the 95% interval, as
  `by_band` gives them) and its drift check, valid 24 hours. With no outcomes reported yet, the
  page says so and draws no chart.

## Data it keeps

Inline rows are deleted when the call ends; a stored dataset 24 hours after its last use; a fitted
model holds cut points and category values, never rows. Keep your own copy of each customer's
record: the scheduled job sends it (or the day's new rows) each time.

## Cost

One decision per answered case; a fit the first time a record is asked about, and on every
`refit_of` question (1,000 free a month,
then $0.01). A `model_ref` question runs no fit. Refusals and `not_yet` bill no decisions.

<!-- generated:other-journeys (npm run docs:build, from src/core/journeys.ts) -->
## Other journeys

| Journey | Fits when the user… | First call | Skill |
|---|---|---|---|
| Try it | has no data yet, or wants to see an answer and a refusal before using their own | `dg_describe`, then `dg_ask` (a sample's ready-to-run ask) | `datagoat-first-run` |
| Ask your data | has a table of past cases with a yes/no outcome (churned, converted, faulted) and a question about it | `dg_add_dataset` (upload: true, one per source), then `dg_map` (returns the ask; dg_backtest and dg_ask follow; a schedule needs fetch_url sources) | `datagoat-ask` |
| Prove it | must show someone the calls were right, or measure whether acting on them worked | `dg_verify`, then `dg_track_record` (or dg_evidence, whether acting on the calls worked) | `datagoat-prove` |
<!-- /generated:other-journeys -->
