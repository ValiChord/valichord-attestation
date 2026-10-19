# Backlog: judge-model configuration

**Status:** Open. Additive.
**Raised by:** independently, within one week, by three people with no contact with each other —
qshanx (then named Seekers2001; `future-agi#1368`, 2026-08-16), Hawthorn (`future-agi#1368`, 2026-08-17),
KeilerHirsch (`lm-evaluation-harness#3749`, 2026-08-19).
**Related:** 03, 04. Same convergence, same `meta` trap (see `README.md`).

## Problem

LLM-as-judge evaluation is now common and the format has nowhere to record the judge. Which model
graded, at what temperature, with what system prompt, at what version. Two runs of the same task
over the same samples, graded by different judges, produce different numbers for reasons the
bundle cannot express.

Three people building against this format named it in the same week, without seeing each other's
comments. That is the strongest evidence available that this is a real gap rather than one
reviewer's taste.

## Evidence from outside this format (added 2026-10-19)

**`UKGovernmentBEIS/inspect_ai#4136`** (opened 2026-06-04 by hiroki-tamba-research, still open, no
maintainer reply as of 2026-10-19) shows the gap in a widely used harness. Inspect's model-graded
scorers call the grader with no generation settings, so the provider default applies: for OpenAI
chat models that is temperature 1.0, unseeded. Identical (output, criterion) pairs then get
different grades on borderline items. Temperature 0 narrows this but does not remove it, and some
grader models ignore temperature altogether. KeilerHirsch joined that thread on 2026-08-14 to frame
the grader as part of the **measuring instrument**. With hiroki-tamba-research he separated random
instrument variance (#4136) from systematic instrument bias (answer-order preference, #4304), and
added that a run design must be able to *estimate* each source, not merely name it.

What that adds to the open questions below:

- **Record what was *in effect*, not only what was asked for.** "Temperature not set" (provider
  default) must be distinguishable from "temperature 0". Some providers ignore the setting. This is
  the same requested-vs-observed split as item 01, applied to the judge.
- **A seed is not a guarantee.** Determinism is the provider's to give. A judge block should be
  able to say "repeated grading, N times" and carry the spread, which connects to 05
  (repeatability) and 06 (uncertainty).
- **Random and systematic error are different fields.** A grade spread across repeats says nothing
  about a judge that consistently prefers the first option. If the format ever records judge
  diagnostics, they should not share one "variance" slot.

## v2 position

Nothing. A judge is a model, and the bundle has one `model_id`, which is the model under test.

## Proposed direction

An optional judge block. The specific fields matter less than the decision in the open questions
below. (BRONCO offered to specify this layer; that offer lapsed: see `README.md`.)

## Open questions

1. **`content_hash` or `meta`?** This is the load-bearing question and it is not close: a
   different judge can change the score, so it belongs where `content_hash` can see it. Putting
   it in `meta` would make two runs with different judges compare as equivalent.
2. Is a judge one model or a pipeline? Rubric-plus-model-plus-aggregation is common, which makes
   this overlap 03 and 04. They may be one field set rather than three.
3. Does a judge configuration want its own hash, so a long prompt need not be carried inline?
