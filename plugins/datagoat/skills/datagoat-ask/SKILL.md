---
name: datagoat-ask
description: '"I have a CSV of churned customers", "will this customer churn", "how risky is this loan", "which leads should we call first", "which machines will fault" - use when someone has a table or log of past cases with a yes/no outcome and a question about cases like them. Answers yes/no, level, best-option or ranking questions from past outcomes, on tables, event logs, time series, panels, sensor streams, agent traces and snapshots (a log read as of chosen moments). dg_suggest finds the questions worth trying; one call, dg_ask, returns a chance, its reasons and a signed Verdict per case, or an honest refusal. Not for reading free text, counting, arithmetic, or predicting a number. For scoring many customers'' cases on a schedule, datagoat-product.'
---

# Ask Datagoat

`dg_ask` answers typed questions about cases from a record of past outcomes. It is
deterministic, it shows its reasons, and it refuses when the record cannot support an answer.

Every call has three parts, and returns a fourth:

- **record** (`data`): past cases with a yes/no outcome column, plus `entity_column`
- **questions**: keyed by an id you choose, each with a `type` and an `outcome_column`
- **cases**: the cases to answer about, `{"ids": [...]}` or `{"rows": [...]}`
- **answers**: keyed by the same ids, each with a `state`, a `pattern`, and per case `p`,
  `reasons`, `pattern_match` and a signed Verdict

If you know Jev: same shape (typed questions in, typed answers out), but the input is a record of
outcomes, not state, and `yesno` is the learned counterpart of a Noul. In Datagoat, `state` is the
answer's status.

**Rule you must never break:** never state a chance, level, reason, range, cut point or direction
the response does not contain. Rephrase; never author. Quote numbers from `text` or the number
fields, never round them further in a way that changes which side of a line a value is on. Say a
case's chance as its `p_display` ("15%", "<1%", ">99%"), word for word, not a rounding of `p` of
your own; in a namespace with a profile, each case's `says` already quotes it. A case the engine
declined (`refused: true`) has no `says`: give its `refused_reason`, not its `p` as a chance.

## Pick the question type

| The user asks | Type | Build it |
|---|---|---|
| Will X happen for this case? How likely? | `yesno` | `{"type": "yesno", "outcome_column": "churned"}` |
| How risky, as a level? | `score` | `{"type": "score", "outcome_column": "defaulted"}` |
| Which option is best for this case? | `choice` | `{"type": "choice", "option_column": "contract", "options": [...], "outcome_column": "churned", "outcome_is_desirable": false}` |
| Which cases first? | `rank` | `{"type": "rank", "outcome_column": "converted", "top_k": 50}` |

`outcome_is_desirable`: `false` for an outcome the user wants to avoid (churned, defaulted),
`true` for one they want (converted, renewed). `choice` requires it: the best option is the
likeliest to give a wanted outcome and the least likely to give an unwanted one. The other types
take it when you know it. Never guess it: if the user has not made it clear, ask.

## Call it once

Put every question about one record in one call. Questions on the same outcome and the same
`outcome_is_desirable` share one fit.
Send cases as `{"ids": [...]}` when they are in the record, or `{"rows": [...]}` when they are new.
For `rank` with no cases, Datagoat ranks the whole record.

## When the data is not one row per case

Pass `time_column` and a `shape`, naming the columns yourself (nothing is inferred), and ask about
cases by id:

| The data is | `shape` |
|---|---|
| an event log (logins, purchases, tickets) | `{"kind": "events", "event_column": "event", "label": {"lapsed": true}, "horizon": {"value": 90, "unit": "days"}}`; the outcome column is then `lapsed_90d` |
| a series with an outcome on each row | `{"kind": "series", "value_columns": [...], "windows": [4, 12]}` |
| periods per case, outcome = a trend | `{"kind": "panel", "label": {"trend_of": "usage", "direction": "down"}}` |
| sensor readings plus fault intervals | `{"kind": "signals", "signal_columns": [...], "event_column": "event_type", "event_start_column": "event_start", "snapshot_every": "1d", "windows": [1, 3, 7], "horizon": 3}` |
| a log of agent runs with a 0/1 outcome | `{"kind": "traces", "agent_column": "agent", "task_column": "task", "tool_column": "tool"}` |
| an event log, and a question about the next N days asked of every case every period ("who cancels in the next 90 days") | `{"kind": "snapshots", "snapshot_every": "4w", "horizon": {"value": 90, "unit": "days"}, "label": {"lapsed": true}, "event_column": "event"}` (or `outcome_time_column` instead of `label`); the outcome column is then `lapsed_90d`, and `cases: {"open": true}` asks about every case today |
| an event log, judged at moments the user chooses (a deal at each stage) | `{"kind": "snapshots", "snapshots": {"dataset_id": "…"}, "snapshot_id_column": "snapshot_id", "snapshot_time_column": "entered_at", "event_column": "activity"}`; the outcome is a column of the snapshot table, and cases are snapshot ids |

A table with several rows for one case (the same deal at each stage, each with that deal's
outcome) needs `group_column` naming the case, or its held-out quality numbers look better than
they are. The snapshots shape does this itself. When the outcome is blank on some rows (not known
yet), an answer's `quality.record` says how many labelled and open rows the fit was measured on:
report those, never the whole row count.

## Logs, or logs with a table: map them first

When the user has an event log (logins, invoices, tickets), perhaps beside a table of accounts,
and no outcome column yet, `dg_map` (free) builds the record: store each source with
`dg_add_dataset`, then call `dg_map` with `sources` (each a `dataset_id` or `fetch_url`),
`outcome_words` in the user's words ("cancel"), and, when the user has said them, `horizon`
(1, 7, 30, 60 or 90 days) and `snapshot_every` (1d, 1w, 4w).

- **Present the questions, never pick.** The proposal's `questions` holds one entry per slot
  where two or more candidates survive (which column names a case, which event is the outcome,
  which column joins the table). Put each to the user with its `options` and `why`, and send
  their choice back as `answers` (`entity`/`time_column`: `[{source, column}]`, `join`:
  `{table, right_key}`, `outcome`: `{offer_id}`). Never choose an option yourself, even the first
  or the likeliest-sounding; `answers: {}` only when `questions` is empty.
- `status: "empty"`: say what `empty_reason` says the sources lack; `not_runnable`: the chosen
  outcome did not pass its gate at this horizon, so offer the other offers or another horizon
  (a record is read at three horizons at most; never try horizons until one clears).
- A confirm returns `mapping_id`, `dataset_id`, `record` and a ready-to-run `ask` on today's open
  cases: add `subject_kind` (ask the user if it is unclear who the cases are) and send it with
  `dg_ask`. Report the record's counts from `record` only (labelled and open cases, positives).
- A single table needs no mapping; `dg_map` on one still adds preflight's report.
- **Before the ask, the record's own past:** a record built from a log can be walked forward with
  `dg_backtest` (`data.dataset_id` from the confirm, `subject_kind`): a fit at each past cutoff on
  what was known then, graded on what came next beside a naive baseline. It answers pending: poll
  `dg_poll` with its `task_id`, never re-send. Report `decision`, `summary` and each cutoff's
  `state` as returned, check its Verdict (kind `backtest`) with `dg_verify`, and call it history,
  not a call about today's cases (that is the `ask`). A table alone is `dataset_not_built`.
  Guide: https://datagoat.io/docs/backtest.

Guide: https://datagoat.io/docs/map.

`dg_describe` has a ready-to-run ask for each on a free sample. Details: https://datagoat.io/docs/shapes.

## When the user doesn't know what to ask

`dg_suggest` (free, fits nothing) lists a table's yes/no questions. Offer the ones with
`worth_asking: true`, each with its ready `question`. It is a screen: about half of what it passes
is answered, so say "worth trying", not "will answer". When `defined_by` is not empty, tell the user
the outcome is (almost) restated by those columns, so an answer would tell them little. A candidate
not worth asking carries the engine's `code` (e.g. `no_separating_signal`, `prevalence_out_of_band`,
`too_few_positives`) and `facts` (rows, positives, prevalence) beside `why_not`: quote them as given
and add no reason of your own.

## Building a product on it

- **One namespace per customer:** `namespace: "<their id>"` on `dg_ask`. Later calls that take a
  `model_ref` find it themselves. The same data in two namespaces is two fits.
- **Fit once, score often:** keep the `model_ref`. A question `{"type": "yesno", "model_ref": "mr1_…"}`
  fits nothing: send the new cases as `cases.rows` (no `data` needed), or send today's record (same
  shape settings as the fit) and name `cases.ids`. Its answers are the model's as fitted; new
  outcomes change nothing until the user asks with the new record (use `refit_of`).
- **What to send:** every answer lists `model_columns`, the columns its model reads; a scheduled job
  sends those. A case missing some, or holding a value the model never saw, comes back
  `state: "not_scoreable"` naming them (no chance, not billed); the other cases are answered. An events or
  snapshots record is read with the fit's own activity types, so a small daily batch works (a type
  with no events counts 0). A model fitted before 2026-09-24: ask once with its full record first.
- **Lifetime:** answers carry `model_expires_at`. `model_ttl_days` (1 to 365) applies only when a
  fit runs; `dg_extend_model` renews, `dg_delete_model` removes. `model_ref_missing` means expired,
  deleted, or another namespace: ask with the record to fit again.
- **Drift, not the lifetime, says whether a model is still good:** refit on a schedule with
  `refit_of`; on `keep` call `dg_extend_model` on `keep_model_ref`, on `refit` switch to the new `model_ref`, on `abandon`
  stop using the question. `no_check_yet` means nothing has been compared: refit first, never renew
  on it. The lifetime is a privacy limit and a safety net.
- **Say how accuracy was measured:** `quality.held_out.whole_cases_by` means every row of one case
  (a deal's every stage) was held out together; tell the user the accuracy is on cases the model
  never saw.
- **`dg_suggest`:** call its results "questions worth trying", never "questions it can answer";
  offer the `worth_asking` ones (listed first), and for `too_few_patterns` pass on `next_step`.

Guide: https://datagoat.io/docs/build.

## Read `state` first

- `answered`: report `p`, the `level` or the `choice`, and the `reasons`; when the user asks why
  or what the risky cases have in common, add the `pattern` and the case's `pattern_match`.
- `refused`: say the record holds no reliable pattern for this question. Do not retry; the same
  call returns the same answer. Suggest more columns or a different outcome.
- A refused or `not_yet` answer carries facts, never a pick: `have.arms_found` (the candidate
  conditions the search found; a pattern needs 4) and, on a record read from a log,
  `horizons_tried` (each horizon the source was read at, with this question's state there:
  answered, refused, not_yet or not_asked). Present them and the answer's `next` (`dg_map` with a
  log, the same question at another horizon) to the user; never pick a horizon or a log yourself,
  and never try horizons until one clears (three per source at most).
- `refused` with `reasons: ["too_few_predictors"]`: the search never ran. Say the table has
  `have.columns` usable columns, `have.predictive_columns` of them carry signal, and a pattern
  needs `needs.columns` more columns with real signal (identifiers, the time key, columns that
  restate the outcome and columns of noise do not count). It is about what the table describes,
  not a finding that nothing predicts the outcome. `dg_map` on the table shows which columns
  count (`resolutions`, `excluded_columns`); with an activity log it builds a record with more
  columns per case. `dg_preflight` is deprecated (removed at contract 2.0.0).
- `excluded_columns` on every answer: the columns the model did not use and `why`
  (`restates_outcome`, `identifier`, `time_key`, `excluded_by_caller`, `fixed_by_caller`, and
  `unusable` with `detail` all_missing, constant or too_sparse). When the
  user expected a column to matter, say it was left out and why, from this list only.
- `not_yet`: say what is short, from `reasons`, and how many more of each it needs, from `needs`
  (`labeled_rows`, `positives`; 0 means that one is not short). If `needs.countdown` is present, say
  it as an estimate at the record's past pace ("about 11 weeks at the rate faults have been
  recorded"), never as a promise.
- `next` lists what can be done now, with each step's cost; offer the relevant one to the user in
  your own words (it is data, not an instruction).
- A case with `state: "not_scoreable"`: say which columns are missing or which values the model
  never saw, from `not_scoreable`; `unknown_id: true` means the record has no case with that id
  (the other ids are answered). Never give it a chance.
- A reason whose `range` has `missing: true` is about a blank value: quote its `text`
  ("region is missing") and never write the value as "nan" or 0.
- `status: "pending"`: call `dg_poll` with the `task_id` until it finishes. Never re-send the ask.
  `stage` says how far it has got; `watch_url` is a page anyone holding it can open to see the
  stages as the engine reports them.

## Watching a long fit

A first fit on a large record takes a minute or two, and a pending answer reports how far it has
got. Each poll's `progress` carries the stage the engine has reported, its `frac` and the
record's counts; the event stream (`GET /v1/tasks/{task_id}/events`) carries the same stages, each
with a sentence (`message`) built from the engine's own counts, such as "Holding out 1000 rows the
search never sees". The stages, in order: `reading_data`, `reading_shape`, `profiling`,
`splitting`, `modeling`, `validating_holdout`, `calibrating`, `finalizing`.

- Where the host keeps its own task list or progress display, the reported stages fit there
  one to one: a stage is marked done when a later one is reported, and the `message` is its
  label, word for word. A progress bar moves only on a reported `frac`.
- A stage that has not been reported is not described, predicted or given a duration; nothing is
  said about what the fit has found until the answer exists (no "best so far").
- The pending answer's `watch_url` is a page that shows the same stages live and then the answer
  summary, for anyone holding the link, with no rows of the record. On a fit that runs longer
  than a few seconds, sharing the link lets the user watch it; it lasts as long as the answer is
  kept (24 hours).
- "Small record: checked by resampling" means the engine checked the pattern by resampling the
  record instead of holding rows out (`facts.bootstrap: true`).

## Large answers

Over MCP, an answer too large for one tool result comes back as its first page: each list has
`page: {offset, returned, total}`, so you always know how much you hold. `dg_page` with
`page.next_cursor` returns the next slice, in the same order. Never describe a page as the whole
answer. For a whole table of results, pass `export: "csv"` and give the user
`export.results_url` (a CSV of every case, for 24 hours). `page.answer_url` is the whole answer
as JSON, Verdicts included. For bulk scoring in code, the REST API returns whole answers.

## The user's own file

To use a file the user has, call `dg_add_dataset` with `upload: true` and give them
`upload_page`, a link to open in their browser (CSV up to 250 MB; the link lasts one hour). In a
host that shows MCP Apps (Claude, ChatGPT), the same call also shows the upload card, where they
choose the file in the conversation and the card reports the `dataset_id`. In ChatGPT, a file the
user attached to the conversation can go straight in as `dg_add_dataset`'s `file`. When they say
it's done, ask about the returned `dataset_id`. Small tables can go inline as `rows` or
`csv` instead. Code that can reach only `api.datagoat.io` (a sandbox) sends the file in pieces to
`upload_pieces_url`; the SDKs' `upload_file` / `uploadFile` and `datagoat upload FILE` do that.
A `.csv.gz` or Parquet file works everywhere a CSV does; Datagoat does not read it, so
`dg_suggest` cannot list its questions (`profile_unavailable`): ask with the columns the user names.

## Report reasons and the pattern faithfully

Each reason names a column, the case's value, `likelihood_direction` (`higher` or `lower`),
`strength` (`strong` or `moderate`) and `range`: the band the value fell in, with a ready `text`
("tenure_months 17 to under 26"). Say it as "tenure of 21 (17 to under 26) is strongly associated
with a higher chance", never "causes". `higher` is not "bad": whether it is good news depends on
the outcome. Keep the order given, and add no reason the response lacks.

`pattern` (yesno, score, rank) lists the conditions under which the outcome is most common in the
record, each with `text` ("support_tickets over 3"). `pattern_match` says how many of them a case
meets and which (`met`). Present the pattern as a description of a group ("churn is most common
when …"), never as a rule or a cause. It is a few conditions, and the chance uses more columns, so
explain one case's number with its reasons, not the pattern. A reason's range and a pattern's cut
point on the same column can differ (tenure 17 to under 26 vs tenure 13 or less); both are
correct, so quote each as given. `choice` answers have no pattern.

With a `shape`, features are built by the reading, and their names say what they measure:
`visit_count_30d` (events: last 30 days), `late_deliveries_max_4` (series: last 4 periods),
`support_tickets__mean` (panel: across all periods), `vibration_max_7` (signals: last 7
snapshots), `tool_prior_outcome_rate` (traces: earlier runs only). Say the window in its unit;
`dg_describe` lists how each shape names them (`shapes[].features`).

## Did acting work?

When an answered case has `levers` with a `lever_token` and the user acts on one, record it with
`dg_attest` (the token, the feature's new value, when). Report outcomes as they arrive with
`dg_report_outcomes`. `dg_evidence` then compares cases acted on with cases not acted on; `live`
is null until each group has 30 outcomes. Report `compliant` and `dose_fraction` as returned, and
never describe the comparison as proof of cause.

## Before anyone acts

Verify every Verdict with `dg_verify`, or say it is unverified. For people
(`subject_kind: "person"`) set `acknowledge_decision_support: true` and tell the user a human
must review the decision.

## Cost

One decision per answered case. Refusals and `not_yet` bill no decisions; verification and the samples are free. The
first ask on a record also runs a fit, whether it answers or refuses; later asks on the same record reuse it.
Asking about the user's own data needs a card on file. A `402 payment_required` means none is on
file yet: tell the user to add one at https://datagoat.io/billing, and don't retry. The samples need no card.

## Related skills

`datagoat-first-run` (a tour on the samples), `datagoat-product` (a product that scores on a
schedule, and keeping it healthy), `datagoat-gate` (acting on an answer, with a person where it
should be), `datagoat-prove` (verifying Verdicts and grading calls against outcomes). All at
https://datagoat.io/skills/<name>/SKILL.md.

## Docs

https://datagoat.io/docs/introduction (start), /docs/ask-your-data, /docs/map, /docs/questions, /docs/answers,
/docs/shapes, /docs/build (ship a product), /docs/patterns, /docs/limits, /docs/ask (every field),
/docs/errors (every error code).

## Try it free

`data: {"dataset_id": "sample:saas_churn"}`, `entity_column: "customer_id"`, `subject_kind: "org"`,
`outcome_column: "churned"`, `cases: {"ids": ["cust_0001"]}`. `dg_describe` lists all samples,
at least one per shape.

## Data

Inline rows are deleted when the call ends; stored datasets 24 hours after last use
(`dg_delete_dataset` deletes one now); models when they expire (`dg_delete_model` deletes one now). Never send health information, card or bank numbers,
government IDs or credentials.

<!-- generated:other-journeys (npm run docs:build, from src/core/journeys.ts) -->
## Other journeys

| Journey | Fits when the user… | First call | Skill |
|---|---|---|---|
| Try it | has no data yet, or wants to see an answer and a refusal before using their own | `dg_describe`, then `dg_ask` (a sample's ready-to-run ask) | `datagoat-first-run` |
| Ship a product | will score many customers' cases repeatedly, on a schedule, inside their own product | `dg_ask` (with namespace and model_ttl_days), then `dg_ask` (by model_ref, no fit) | `datagoat-product`, `datagoat-gate` |
| Run it | already has a model_ref in use and is learning what happened to the cases it scored | `dg_report_outcomes`, then `dg_schedule` (or refit_of + dg_drift by hand) | `datagoat-product` |
| Prove it | must show someone the calls were right, or measure whether acting on them worked | `dg_verify`, then `dg_track_record` (or dg_evidence, whether acting on the calls worked) | `datagoat-prove` |
<!-- /generated:other-journeys -->
