---
name: datagoat-gate
description: Use when an agent or workflow will ACT on a Datagoat answer - send an offer, open a ticket, reassign a lead, block a run, message a customer - and must decide what it may do alone, what needs a person, and what to leave. Builds the gate - state first, then the engine's band (act, escalate, refuse) and max_autonomy, then your own thresholds - and places the human approval before the side effect so a retry never acts twice. Examples for AgentFactory, LangGraph and n8n. For asking the question itself, use datagoat-ask.
---

# Gate an action on a Datagoat answer

Datagoat supplies the chance, its reasons and a signed Verdict. The decision to act, and what a
mistake costs, are yours. Docs: https://datagoat.io/docs/agents.

**Rules you must never break**

- Act only on `state: "answered"`. `refused` and `not_yet` mean no action from this answer: take
  the fallback path, and never retry a refusal.
- Verify the Verdict (`dg_verify`, or the SDK's offline verifier) before any action a person
  could not undo.
- Never state or act on a number the answer does not contain.
- Questions about people (`subject_kind: "person"`) are decision support: a person reviews every
  decision, whatever the band says.

## The gate, in order

1. **State.** Not `answered` (`decision` is not `actionable`) → fallback. Stop. A case marked
   `not_scoreable` has no chance → fallback for that case.
2. **Band.** Ask with `band: true`. Each case then carries:
   - `band`: `act` (the engine would stand behind acting), `escalate` (uncertain: a person should
     look), or `refuse` (do not act on this case; `band_reason` says why, e.g. `low_likelihood`).
     A `refuse` band on an answered case is advice about that case, not a refused answer.
   - `max_autonomy`: the most the agent may do alone: `L0` observe only, `L1` suggest, `L2` act
     after a person approves, `L3` act, then a person reviews.
3. **Your threshold.** Within what the band allows, compare `p` with a threshold set by what a
   wrong action costs: cheap and reversible → a lower bar; expensive or permanent → a higher bar,
   or always a person.
4. **Human step before the side effect.** Pause, show the person the case, `p`, the reasons with
   their `range.text`, and the Verdict's `verdict_id`; act only on approval. Give the action an
   idempotency key derived from the Verdict (`verdict_id` + action name) so a retry after the pause
   cannot act twice.
5. **Record it.** If you acted through a case's lever, `dg_attest` with its `lever_token` and an
   `event_id`; when the outcome is known, `dg_report_outcomes`. See `datagoat-prove`.

| State | Band | `max_autonomy` | The agent may |
|---|---|---|---|
| refused / not_yet | — | — | nothing from this answer; fallback |
| answered | refuse | L0 | nothing for this case |
| answered | escalate | L1 | draft a suggestion for a person |
| answered | act | L2 | act after a person approves |
| answered | act | L3 | act, then queue for review |

## AgentFactory

`nango.http@5` → `POST /v1/ask` on the `datagoat` connection with `"band": true`; a `logic.router`
on `steps.ask.output.body.answers.<id>.state`, then on
`steps.ask.output.body.answers.<id>.cases[0].band`; `act` with L2 goes through `human.approval`
before the step that acts; `escalate` posts the case and its reasons to a channel; everything else
ends quietly.

## LangGraph

```python
out = dg.ask(questions, band=True, **record)
a = out["answers"]["churn"]
if a["state"] != "answered":
    return fallback()
case = a["cases"][0]
if case["band"] == "act" and case["max_autonomy"] in ("L2", "L3"):
    decision = interrupt({"case": case["entity_id"], "p": case["p"], "reasons": case["reasons"]})  # pause BEFORE acting
    if decision == "approve":
        act(case, idempotency_key=f'{a["verdicts"][0]["verdict"]["verdict_id"]}:offer')
elif case["band"] == "escalate":
    notify_reviewer(case)
```

## n8n

HTTP Request (`POST https://api.datagoat.io/v1/ask`, `band: true`) → IF on `state` → Switch on
`band` → a Wait node ("resume on webhook") for approval → the acting node, whose request carries
the idempotency key.
