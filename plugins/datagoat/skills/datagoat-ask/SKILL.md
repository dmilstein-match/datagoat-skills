---
name: datagoat-ask
description: Use whenever an agent must answer a yes/no, level, best-option or ranking question about cases from past outcomes — will this customer churn, how risky is this loan, which contract or team is best for this case, which leads to call first, which machines will fault, which agent runs will fail. Works on tables, event logs, time series, panels, sensor streams, agent traces and snapshots (a log read as of chosen moments), and for products that score on a schedule per customer. One call, dg_ask, returns a chance, its reasons and a signed Verdict per case, or an honest refusal. Not for reading free text, counting, arithmetic, or predicting a number.
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
fields, never round them further in a way that changes which side of a line a value is on.

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
| an event log, judged at moments the user chooses (a deal at each stage) | `{"kind": "snapshots", "snapshots": {"dataset_id": "…"}, "snapshot_id_column": "snapshot_id", "snapshot_time_column": "entered_at", "event_column": "activity"}`; the outcome is a column of the snapshot table, and cases are snapshot ids |

A table with several rows for one case (the same deal at each stage, each with that deal's
outcome) needs `group_column` naming the case, or its held-out quality numbers look better than
they are. The snapshots shape does this itself.

`dg_describe` has a ready-to-run ask for each on a free sample. Details: https://datagoat.io/docs/shapes.

## When the user doesn't know what to ask

`dg_suggest` (free, fits nothing) lists a table's yes/no questions. Offer the ones with
`worth_asking: true`, each with its ready `question`. It is a screen: about half of what it passes
is answered, so say "worth trying", not "will answer". When `defined_by` is not empty, tell the user
the outcome is (almost) restated by those columns, so an answer would tell them little.

## Building a product on it

- **One namespace per customer:** `namespace: "<their id>"` on `dg_ask`. Later calls that take a
  `model_ref` find it themselves. The same data in two namespaces is two fits.
- **Fit once, score often:** keep the `model_ref`. A question `{"type": "yesno", "model_ref": "mr1_…"}`
  fits nothing: send the new cases as `cases.rows` (no `data` needed), or send today's record (same
  shape settings as the fit) and name `cases.ids`. Its answers are the model's as fitted; new
  outcomes change nothing until the user asks with the new record (use `refit_of`).
- **What to send:** every answer lists `model_columns`, the columns its model reads; a scheduled job
  sends those. A refusal (`row_not_scoreable`) names every missing column at once. An events or
  snapshots record is read with the fit's own activity types, so a small daily batch works (a type
  with no events counts 0). A model fitted before 2026-09-24: ask once with its full record first.
- **Lifetime:** answers carry `model_expires_at`. `model_ttl_days` (1 to 365) applies only when a
  fit runs; `dg_extend_model` renews, `dg_delete_model` removes. `model_ref_missing` means expired,
  deleted, or another namespace: ask with the record to fit again.
- **Drift, not the lifetime, says whether a model is still good:** refit on a schedule with
  `refit_of`; on `keep` call `dg_extend_model`, on `refit` switch to the new `model_ref`, on `abandon`
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
- `not_yet`: say how many more labeled rows it needs (`needs.labeled_rows`).
- `status: "pending"`: call `dg_poll` with the `task_id` until it finishes. Never re-send the ask.
  `stage` says how far it has got.

## Large answers

Over MCP, an answer too large for one tool result comes back as its first page: each list has
`page: {offset, returned, total}`, so you always know how much you hold. `dg_page` with
`page.next_cursor` returns the next slice, in the same order. Never describe a page as the whole
answer. For a whole table of results, pass `export: "csv"` and give the user
`export.results_url` (a CSV of every case, for 24 hours). `page.answer_url` is the whole answer
as JSON, Verdicts included. For bulk scoring in code, the REST API returns whole answers.

## The user's own file

To use a file the user has, call `dg_add_dataset` with `upload: true` and give them
`upload_page`, a link to open in their browser (CSV up to 250 MB; the link lasts one hour). When
they say it's done, ask about the returned `dataset_id`. Small tables can go inline as `rows` or
`csv` instead.

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

https://datagoat.io/docs/introduction (start), /docs/ask-your-data, /docs/questions, /docs/answers,
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
