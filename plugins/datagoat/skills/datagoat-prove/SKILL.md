---
name: datagoat-prove
description: Use when someone needs to show that Datagoat's answers were right, or that acting on them worked - "prove it", "did the model call it", "show the track record", "audit these decisions", "did the retention offers work". Covers verifying Verdicts (with dg_verify or offline, with no call to Datagoat), keeping each signed call from before the outcome, grading calls against what happened in a walk-forward replay, recording actions with dg_attest and outcomes with dg_report_outcomes, reading dg_track_record (the model's calls against reported outcomes: calibration by band, level and chance) and reading dg_evidence without claiming cause.
---

# Prove it

A claim about predictions is only as good as its timestamps. Datagoat signs every answer when it
is made, so a record of calls can be checked by anyone later. Docs: https://datagoat.io/docs/prove.

**Rules you must never break**

- Never state a chance, count or rate that neither a response nor the user's own outcome data
  contains; say where each number came from.
- Evidence from acting is observational: cases acted on were chosen, not randomised. Never call it
  proof of cause.

## 1. Every call is signed

Each answer carries `verdicts`: a `verdict` (the chances, reasons, `computed_at`, `expires_at`,
the record's `dataset_content_hash`, the engine's `core_hash`) and a `signature`.

- `dg_verify` (free, no key over REST) returns `valid`, `invalid_signature`, `expired` or
  `unknown_key`. Pass the verdict and signature exactly as received.
- With no call to Datagoat: the SDKs' `verify_offline` (Python) / `verifyOffline` (TypeScript)
  check the Ed25519 signature against https://api.datagoat.io/.well-known/jwks.json. The signed
  bytes are the protected header, a dot, and the verdict as compact JSON with sorted keys.
- Store each Verdict and signature when the call is made. Datagoat keeps answers for 24 hours only.

## 2. Grade calls against what happened (a walk-forward replay)

1. Fit only on history up to a cutoff date (send only rows before it; `time_column` orders them).
2. For each later period, score that period's cases from the saved `model_ref` (no fit) and store
   the signed answer before looking at any outcome after it.
3. After the whole loop, and only then, read the outcomes and grade: of the cases in the top band
   or top-k each period, how many had the outcome, against the rate across all cases that period;
   how far ahead the first call came.
4. Say what the replay does not show: one dataset or many, which outcome, how the cases were
   chosen, and that past calls do not guarantee future ones.

The grading arithmetic is your own counting of outcomes against signed calls; say so when you show
it.

## 2b. The track record Datagoat keeps

`dg_track_record` (free) with the `model_ref` sets the model's earlier calls against the outcomes
reported with `dg_report_outcomes`: each outcome paired with the latest answer about that case given
on or before the day it was observed. Read `overall`, then `by_band`, `by_level` and `by_chance`:
`observed_rate` against `mean_chance` says whether the model's chances held up (near: calibrated),
with `interval_95`. Say `small_n` plainly when it is true (under 100 calls). Say that answers are
kept only from `records_since` on, and never call a match proof that acting caused anything.

## 3. Did acting work?

- When you act on a case through one of its `levers`, record it: `dg_attest` with `model_ref`,
  `entity_id`, the lever's `lever_token` exactly as given, `post_value`, `acted_at`, and an
  `event_id` so a retry is safe. It returns `compliant` and `dose_fraction`; report them as given.
- Report outcomes as they arrive: `dg_report_outcomes` (`{entity_id, outcome, observed_at,
  event_id}` rows).
- `dg_evidence` compares cases acted on with cases not acted on. `live` is null until each group
  has 30 outcomes; `small_n` is true below 100. Report `diff` with its interval, and call it an
  association.

Writing outcomes and attestations needs the "Can report outcomes" capability (a person signed in
through an MCP host has it; an API key has it when created with it).
