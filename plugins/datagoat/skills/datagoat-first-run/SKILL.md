---
name: datagoat-first-run
description: Use when someone wants to see what Datagoat does before using their own data — "show me Datagoat", "try it", "what can it do", "demo it on the samples". Walks the free samples in about five minutes - one answered question with its reasons, one question it refuses and why, the same data answering once an activity log is added, and a Verdict verified - so the person sees an answer, a refusal and a proof. Needs no card and no account beyond a free test key. For their own data, hand over to datagoat-ask.
---

# First run on the free samples

Five minutes, four moments: an answer, a refusal, the refusal turning into an answer, and a proof.
Every number you say comes from a response; never quote one from this page.

**Rule you must never break:** never state a chance, level, reason, range or direction the
response does not contain. Rephrase; never author.

## 0. Connect

- Over MCP (`https://api.datagoat.io/mcp`), the `dg_` tools are already there.
- Without MCP: `POST https://api.datagoat.io/v1/agents/register` returns a free test key
  (`dgk_test_…`) that works on the samples only. Send it as `Authorization: Bearer …`.
- `dg_describe` lists the samples, one per record shape, each with a ready-to-run ask.

## 1. An answer

```json
{"data": {"dataset_id": "sample:saas_churn"}, "entity_column": "customer_id", "subject_kind": "org",
 "questions": {"churn": {"type": "yesno", "outcome_column": "churned", "outcome_is_desirable": false},
               "risk":  {"type": "score", "outcome_column": "churned", "outcome_is_desirable": false}},
 "cases": {"ids": ["cust_0001"]}}
```

Read `answers.churn.state` first. Then say the chance (`p`), the level, and the reasons in the
order given, each with its `range.text`. Reasons are associations, not causes.

## 2. A refusal

Ask whether a deal will be won from the deal table alone:

```json
{"data": {"dataset_id": "sample:deal_stages"}, "entity_column": "snapshot_id", "subject_kind": "org",
 "group_column": "deal_id",
 "questions": {"won": {"type": "yesno", "outcome_column": "won", "outcome_is_desirable": true}},
 "cases": {"ids": ["deal_0001@demo"]}}
```

It comes back `refused`: the stage and the amount hold no pattern that survives on deals the model
never saw. Say that plainly. A refusal is an answer, the same every time; do not retry it, and it
bills no answered cases.

## 3. The same question, with the evidence it needed

Add the activity log (calls, emails, meetings before each stage) through the `snapshots` shape:

```json
{"data": {"dataset_id": "sample:deal_activity"}, "entity_column": "deal_id", "subject_kind": "org",
 "time_column": "ts",
 "shape": {"kind": "snapshots", "snapshots": {"dataset_id": "sample:deal_stages"},
           "snapshot_id_column": "snapshot_id", "snapshot_time_column": "entered_at",
           "outcome_time_column": "closed_at", "event_column": "activity",
           "value_columns": ["attendees"], "lookback_days": [7, 30]},
 "questions": {"won": {"type": "yesno", "outcome_column": "won", "outcome_is_desirable": true}},
 "cases": {"ids": ["deal_0001@demo"]}}
```

Now it answers. Say the chance, the reasons, the `pattern` (where winning is most common) and the
case's `pattern_match`. `quality.held_out.whole_cases_by` means accuracy was measured on whole deals
the model never saw.

## 4. A proof

Take one Verdict from the answer (`answers.won.verdicts[0]`) and call `dg_verify` with its
`verdict` and `signature` exactly as received: `valid`. Change one number in a copy and verify
again: `invalid_signature`. Anyone can do this without Datagoat: the SDKs' `verify_offline` /
`verifyOffline` check the Ed25519 signature against https://api.datagoat.io/.well-known/jwks.json.

## Then

- Their own data: the `datagoat-ask` skill (upload, `dg_suggest`, `dg_preflight`, ask).
- A product on a schedule: `datagoat-product`. An agent that acts on answers: `datagoat-gate`.
- Docs: https://datagoat.io/docs/quickstart
