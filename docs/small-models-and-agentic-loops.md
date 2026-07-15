# Small models and agentic loops: scaffold the loop, keep the judgment

A 12B model was asked to run an adversarial critique: *spawn two parallel sub-agents, apply
their fixes to the document in place, then emit a JSON list of the notes you raised.* It ran
for **41 minutes** and hit the harness timeout. The completed document it was critiquing was
thrown away with it.

The tempting conclusion — *"a 12B can't do agentic work"* — is **wrong**, and acting on it
costs you the cheapest capable model in your fleet. The real lesson is narrower and more
useful.

> Labels: `[PEER-REVIEWED]` · `[MEASURED-RAW]` (primary data we pulled) · `[COMMUNITY-ANECDOTE]`.

## 1. First, the claim that isn't true

**"Small models can't do multi-turn tool loops"** is usually argued from a number that
doesn't say that. `[MEASURED-RAW]` — from the
[BFCL raw data](https://gorilla.cs.berkeley.edu/data_overall.csv):

| Claude-Haiku-4-5 | Overall | Single-turn AST | **Multi-Turn** |
|---|---|---|---|
| **Function-calling (tools)** | 68.70% | 86.50% | **53.62%** |
| **Prompt-only (no tools)** | 25.26% | 55.42% | **1.75%** |

The infamous **1.75%** floor is **prompt-mode — tools removed**. With native tools, multi-turn
is **53.62%**. Those are different universes, and citing the first against a *tool-using* loop
is a category error. (We made it ourselves before checking.)

And in our own probe `[MEASURED-RAW]`, **gemma-4-12b-qat emitted native tool calls 3/3** on a
write task. It can call tools. One timed-out run, unscaffolded, against an unbounded 30-minute
wall, is not evidence of incapability.

## 2. What the evidence actually supports: scaffolding

`[PEER-REVIEWED]` **Agents' Room**, [arXiv:2410.02603](https://arxiv.org/abs/2410.02603)
(ICLR 2025, Google DeepMind + Edinburgh) is the closest controlled test that exists. It
decomposes narrative writing into subtasks handled by specialized agents with an orchestrator
sequencing them, against an **end-to-end baseline where the same backbone writes the whole
thing directly**. Backbone held constant: **Gemini 1.5 Flash — a small model**. Human raters,
all unordered pairs. Result: the scaffolded variants **beat end-to-end across every dimension**
(plot, creativity, development, language use).

**And the confound runs in the small model's favor:** the weaker the backbone, the more a
scaffold helps. Scaffolding is worth **most** at 12B, not least.

Caveat worth stating: one team, one scaffold, one backbone, built to validate that scaffold —
and nobody re-runs it. It is a demonstration, not a leaderboard. But it is the right shape of
evidence, and it points the opposite way from "small models can't."

## 3. The actual bug: the model was doing two jobs at once

Re-read the failing task. The 12B was asked to simultaneously:

| | |
|---|---|
| **orchestrate** | spawn parallel sub-agents, sequence them, decide when to stop |
| **mutate state** | apply fixes to the document in place |
| **judge prose** | evaluate the writing and raise notes |
| **serialize** | emit conforming JSON |

Three of those four are the things small models are worst at. **One of them — judging prose —
is something they're genuinely good at.** We fused all four into a single turn and then blamed
the model for the one it couldn't carry.

## 4. The fix: deterministic orchestrator, model as critic

Split them along the line the evidence draws:

```
Python orchestrator        fans out N focused critique calls, one per lens
  → the 12B (single-shot)  reads the doc, returns notes for THAT lens        ← its strongest regime
Python                     collects notes, applies edits, decides convergence
```

Why each piece lands where it does:

- **The model does single-shot judgment.** BFCL single-turn: prompt-mode **89.65%** actually
  *beats* function-calling's 88.58%. Take the tools away and a single focused pass is the
  regime where small models are strongest — not weakest.
- **Python does orchestration.** Fan-out, sequencing, and termination are deterministic
  problems. Code doesn't time out deciding whether it's done.
- **Python does the effects.** No in-place edit means no
  [empty-patch](./adherence-degrades-first.md) failure mode, and nothing to leave
  half-written.
- **Nothing to time out.** A completion ends at EOS. The 41-minute wall existed only because
  the model was steering an open-ended loop.

Same critique work. Still mostly the 12B. No sub-agent spawning, no in-place mutation, no
unbounded turn.

**Do NOT reach for a grammar to force the JSON here** if the loop still has tools — a schema
constraint alongside `tools` suppresses tool-calling **entirely**
([6/6 → 0/6, measured](./format-tax-and-generate-then-conform.md#we-reproduced-it-measured--our-own-fleet-2026-07-15)).
Drop the tools (single-shot capture) and the conflict disappears.

## 5. Local judging is already cheap enough to mean it

`[BENCHMARK-MEASURED]` [EQ-Bench Judgemark v4](https://eqbench.com/) — measuring whether a
model can *discriminate* writing quality:

| Judge | Score | Cost |
|---|---|---|
| claude-opus-4-6 | 0.907 | $39.37 |
| gpt-5.5 | 0.878 | $30.44 |
| **google/gemma-4-31B-it** | **0.723** | **$0.82** |
| Qwen/Qwen3.5-27B | 0.605 | $1.76 |

A local Gemma-4-31B judges at **~80% of frontier separability for ~1/48th the cost**, beating
gpt-5.4 (0.721) and claude-sonnet-5 (0.709) — CIs overlap heavily, so read it as "competitive,"
not "better." **Caveat that matters:** Judgemark measures *separability*, not *correctness*. A
judge can be sharply discriminating and consistently wrong, and Judgemark cannot tell you which.

Still: the critic doesn't have to be expensive. It has to be **focused**.

## 6. When to actually reach for a bigger model

Scaffolding is not universal absolution:

- **Genuinely unbounded exploration** (research, open-ended web navigation) — tool-choice *is*
  the reasoning. Prompt-mode floors at 1.75% for a reason. Use tools, on a model that can.
- **When the content underneath is weak.** CODESTRUCT's Qwen3-8B recovered 41 patches and
  gained **zero** accuracy. A scaffold surfaces the model's judgment; it doesn't improve it.
- **Don't over-rotate into anti-agent ideology.** The flagship anti-agent result's own authors
  defected: Xia & Zhang's [Live-SWE-agent](https://arxiv.org/abs/2511.13646) scores **77.4%**
  vs Agentless's **50.8%**.

## 7. Takeaways

1. **Check which harness a benchmark number came from.** The multi-turn "floor" everyone cites
   is prompt-mode; with tools the same model is 30× better.
2. **Don't ask one turn to orchestrate, mutate, judge, and serialize.** Split by what each side
   is good at — that fusion, not the model, is usually the bug.
3. **Put the loop in code and the judgment in the model.** A weak backbone benefits *most* from
   a scaffold.
4. **A timeout is not a verdict on capability.** It's a verdict on the harness that let an
   open-ended turn run unbounded.
