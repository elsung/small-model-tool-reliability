# Instruction adherence: order effects and freedom-scaled judging

The rest of this repo is about the generation *mechanism* — making a small model reliably call a
tool, or reliably emit well-formed structured output, instead of narrating or rambling. This
document is about a different, downstream failure: even when the model *is* producing well-formed
output for a constrained-transform task, does that output actually **honor the constraint**? Two
field findings, both from a real constrained-transform pipeline (rework source-derived narrative
content into a new setting, with an explicit instruction that none of the source's own identifying
names may survive the transform), sanitized and generalized.

## Contents

- [1. Instruction order controls adherence, decisively](#1-instruction-order-controls-adherence-decisively)
- [2. Freedom-scaled adherence judging](#2-freedom-scaled-adherence-judging)

---

## 1. Instruction order controls adherence, decisively

### The incident

A small model tasked with a constrained transform — informally, "rewrite this content into a new
setting; invent all-new character names; none of the source's own names may appear anywhere in the
output" — leaked the source material's own proper nouns into its output at wildly different rates
depending **purely on where the identity/rename instruction sat in the prompt**, with model,
sampler, and source content held constant across 9 measured runs:

| Instruction placement | Result |
|---|---|
| Rename/identity transform **first**, fused to the primary verb, phrased as one absolute-scope clause ("Transform X into Y — invent all-new names; none of the originals may appear anywhere") | **0 leaks** |
| The *same* requirement **last**, after a stack of world/lore negations (setting/genre prohibitions) | **76–184 leaked instances** of the source's own names |

Same model, same underlying requirement, same source — the only variable that moved was where one
clause sat in the instruction stack, and it was the difference between a clean pass and a
double-digit-to-triple-digit leak count.

### Why this happens

The model did **not** fail uniformly at holding instructions under load. Across every run, it held
every *world/lore* negation correctly, **regardless of where that negation sat in the prompt** — a
prohibition like "don't use terminology from the source's genre/setting" was respected whether it
appeared early or late. What changed was specifically **identity preservation**: the model appears
to treat "keep the source's names" as its *default disposition* unless renaming is the **primary,
headline command** — buried at the end of a list of other negations, the rename instruction reads
to the model as one more item in a checklist rather than the actual job, and the model falls back to
its default (preserve identity) rather than executing it as the main transform.

This is a genuinely useful, counterintuitive finding: **different constraint *types* get different
positional robustness from a model's training, and it is not safe to assume uniform recency or
primacy decay across an entire instruction stack.** Some constraint categories (in this measurement,
world/lore prohibitions) are apparently reinforced strongly enough in training that the model checks
them regardless of position. Others (in this measurement, identity/naming) are treated as a soft
default that only gets overridden when they're the headline instruction, fused to the primary verb —
buried, they get skipped in favor of the default behavior, not because the model "forgot," but
because it never registered the clause as the actual task.

**A prior theory was tested and disproven, worth recording because it's a plausible-sounding wrong
explanation:** the working hypothesis going in was "evoking a genre pulls in that genre's own
well-known proper nouns as a kind of contamination" — i.e., the leaks would be recognizable IP names
from the *target* genre bleeding in unbidden. Measurement showed this was **wrong**: zero
genre/IP-associated names leaked in any run. **100% of the leaked strings were the source
material's own names** — the input's identity, not some third genre's identity — and the leak rate
tracked instruction *position*, not genre framing, cleanly across all 9 runs. The lesson isn't just
the specific finding — it's that the intuitive-sounding explanation ("evoking X pulls in X's
canon") was measured and falsified, and the real cause (pure instruction-order effect) would never
have been found by reasoning about it in the abstract. **Measure, don't theorize** — a recurring
theme across both repos in this pair.

### The fix

- **Lead with the identity/rename transform**, fused to the primary verb, as a single absolute-scope
  clause: name the transform and the "nothing survives" scope in the same sentence, not as two
  separate instructions.
- **Follow with the world/lore exclusion lists after** — those hold up regardless of position, so
  they don't need the same positional privilege.
- **Structure of the instruction beats its length or exhaustiveness.** Adding more negations, more
  examples of what not to do, or more emphasis ("IMPORTANT: do not reuse names") did not fix the
  problem when the rename instruction was buried — reordering it to be first and fused to the verb
  did, immediately, and completely (0/9 leaks in that configuration vs. leaks in every other tested
  ordering).
- **Generalizing:** for any constrained-transform task, don't assume every constraint in your prompt
  gets equal positional treatment. Identify which constraint categories your model treats as
  "default-preserve unless it's the headline command" (identity/naming, in this measurement) versus
  "enforced correctly regardless of position" (world/lore prohibitions, in this measurement) — this
  will vary by model and by constraint domain, so treat it as something to *measure* per model
  family and per constraint type, not assume transfers. Put whichever categories are
  default-preserve **first**, fused to the primary verb, and don't bury them under other
  instructions no matter how logical the checklist order feels to a human reader.

---

## 2. Freedom-scaled adherence judging

### The problem

Once you have a model reliably attempting a constrained-transform task, the natural next move is an
automated adherence check — a judge or gate that flags output that's drifted from what the
instructions asked for, so a human doesn't have to manually review every run. The naive version of
this judge — pause or flag on **any** detected deviation from the source — is actively worse than no
judge at all: real transform jobs are *supposed* to deviate from the source along whatever axes the
job explicitly invited (new setting, new character names, reworked scenes), and a judge that treats
every one of those invited deviations as a violation generates constant false alarms. Users
learn, fast, to ignore or auto-dismiss a noisy judge — and once that habit forms, the one run with a
**real**, load-bearing violation gets dismissed along with all the noise. A judge nobody trusts is
worse than no judge, because it creates false confidence that review is happening when it's
actually being rubber-stamped.

### The fix

Classify every element of the instruction set into one of two buckets before judging anything:

- **HARD** — keep/forbid rules and core-intent requirements that must hold **regardless of how much
  creative freedom the job otherwise grants**: things explicitly named as must-keep, must-not-appear,
  or the load-bearing point of the job (e.g., in the incident above, "the source's own names must not
  appear" is HARD — it's the actual instruction, not a stylistic preference).
- **SOFT** — elements the job explicitly invites the model to transform, reimagine, or depart from
  (setting, phrasing, specific scene staging, tone shifts that were asked for).

Pause the pipeline (or flag for human review) **only** on a HARD-constraint violation, or on
total-intent abandonment (the output no longer serves the job's actual goal at all — not "changed a
lot," but "isn't doing the job anymore"). Scale the tolerance for total-intent judgments to how much
freedom the job actually granted: a job that invited "bold, expansive departure" should tolerate
far more surface change before anything gets flagged than a job that asked for close fidelity to a
source. **Bold transformation of an invited (SOFT) element is success, not drift — the judge should
never flag it, no matter how large the change looks against the original source**, because "look how
different this is from the source" is exactly the outcome that kind of job was asked to produce.

### Why this connects to the sibling repo's freedom lever

This HARD/SOFT split is the judging-side counterpart of the `freedom_lever` concept from the sibling
[llm-context-management](https://github.com/elsung/llm-context-management) repo's goal triple
([docs/04 §4.2](https://github.com/elsung/llm-context-management/blob/main/docs/04-goal-aware-extraction.md#42-the-goal-triple--what-parameterizes-relevance)):
that repo defines `directives` as a **fidelity floor** for context reduction (an instruction pins
something at high fidelity so a reducer can't compact it away) and `freedom_lever` as the dial that
governs how aggressively everything *not* pinned can be compacted. **A HARD constraint here is
exactly a directive-pinned fidelity floor; a SOFT element is exactly what the freedom lever licenses
the model to depart from.** The practical implication: an adherence judge and a context-reduction
layer sitting in front of the same generation job should read the **same** `freedom_lever` value and
the **same** directive list, rather than each subsystem defining "how much freedom is this job
allowed" independently and inconsistently — a job dialed to `expansive` for reduction purposes but
judged as if it were `faithful` (or vice versa) will produce exactly the false-alarm noise (or,
worse, the false confidence) described above, just from the two halves of the pipeline disagreeing
about the same job-level setting.

That sibling repo also documents a **complementary small-model behavior worth knowing alongside
this one**: at the `faithful` end of the same freedom lever, small models tend to
**under-transform** — doing the smallest edit that could plausibly be called a change, rather than
genuinely re-expressing material within the fidelity band intended
([docs/04 §4.6](https://github.com/elsung/llm-context-management/blob/main/docs/04-goal-aware-extraction.md#46-a-small-model-specific-failure-mode-under-transformation-at-the-faithful-end)).
Read together, §1 of this document and that finding describe two ends of the same axis: pushed
toward `faithful` with the identity/rename instruction under-emphasized, a small model **fails to
transform enough** (under-transformation) or **fails to stop preserving what should have been
replaced** (the leak failure in §1) — both are the model defaulting to "preserve the source" when
its actual instructions should have overridden that default. A freedom-scaled judge, driven by the
same lever value, is the safety net that catches both directions of that same underlying failure
mode.

## Related work

This document is part of **[small-model-tool-reliability](https://github.com/elsung/small-model-tool-reliability)**,
which otherwise covers generation-*mechanism* reliability (tool calls, structured output, sampler
safety). Its sibling repo, **[llm-context-management](https://github.com/elsung/llm-context-management)**,
covers keeping a pipeline's *input* inside a real context budget, including the `freedom_lever` and
directive/fidelity-floor concepts this document builds on. See that repo's
[docs/04-goal-aware-extraction.md](https://github.com/elsung/llm-context-management/blob/main/docs/04-goal-aware-extraction.md)
for the full goal-triple model.
