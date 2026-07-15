# A tool-calling reliability matrix across small local models

The [main writeup](../README.md) shows *one* model family (a Gemma-4-12B-class model) narrating tool
calls ~25% of the time on a real multi-tool write loop, and grammar-constrained decoding fixing it.
This document widens that to a **matrix across model families and sizes served on the same box and
build** — so the "which models actually need the grammar, and which are fine natively" question is
answered empirically, not by reputation.

The point is not the specific percentages (they will vary with your build, quant, prompt, and
temperature). The point is the **shape of the finding**: reliability tracks the *serving tool-parser
maturity of the model family*, not model size. A 3B model with a mature native tool parser can beat a
31B model with a fragile one.

> **Second axis — quantization.** This matrix holds *within a fixed quant*. There is an independent
> degrader: **aggressive quantization (sub-Q4: IQ2_XXS / Q2_K) erodes tool-calling even on a
> native-reliable parser family** — the same big MoE that tool-calls flawlessly at FP8 can flail and
> never terminate the turn at IQ2_XXS. So the full rule is *needs-grammar = fragile parser family **OR**
> aggressive quant*. See [the quant axis](./quantization-and-tool-reliability.md).

## Test method

Identical harness for every model:

- **Native** — the real agent CLI, **6 tools** (`read, find, grep, write, edit, bash`), a non-inlined
  two-step **read → compose → write** loop, `thinking: medium`, a realistic multi-thousand-token brief,
  temperature 0.9. A trial is a **flake** if no file is written (the model narrated "written to …",
  or wrote a refusal stub) — measured over N≥15 real runs. Tested with `tool_choice: auto` (the real
  default) and, separately, `tool_choice: required`.
- **Grammar** — the same artifact via a **direct call with a loose GBNF grammar** (force the section
  skeleton, permissive body) + a bounded `max_tokens`. A trial is a flake if the output is not
  valid structured content. N≥15.

> A methodological warning worth repeating: a **simplified curl bench (1 tool, single turn) does NOT
> reproduce the flake** — it reported 0% on the very model that flakes 25%+ on the real 6-tool loop.
> You have to measure the *real agentic shape* (multiple tools, a genuine read→write loop, agent
> `thinking` on) or you will conclude, wrongly, that everything is fine. Measure the loop you actually run.

## The matrix

`native (auto)` / `native (required)` = no file written (narrated, refusal stub, or tool JSON the parser
rejected → `rc≠0`) over N real agent-CLI runs. `grammar` = output not opening at the forced structural
heading, over N direct grammar-forced calls. Reduced prompt scale (short artifact) for the live sweep so
every model could run concurrently; the fragile-parser model's headline flake is from the **full-scale**
prompt (see the load-dependence note).

| model | params / arch (parser family) | native flake (auto) | native flake (required) | grammar flake | verdict |
|---|---|---|---|---|---|
| **Qwen3.5-9B** | 9B dense (Hermes/native) | **0%** (0/12) | **0%** (0/10) | ~0% (15/15) | **works natively** |
| **Qwen3-14B** | 14B dense (Hermes/native) | **0%** (0/12) | **0%** (0/10) | ~0% (15/15) | **works natively** |
| **Gemma-4-12B-QAT** | 12B dense QAT (fragile family parser) | 0% at reduced scale · **~25%** at full scale (N=16) | 0%† (not enforced) | ~0% (15/15) | **needs grammar** (load-dependent) |
| **Nanbeige4.1-3B** | 3B (thinking) | **25%** (3/12) | — | ~0% (15/15) | **needs grammar** |
| **Llama-3.2-7B creative finetune #1** | 7B creative/roleplay finetune | **100%** (12/12, parser-reject) | — | ~0% (15/15) | **needs grammar / avoid native tools** |
| **Llama-3.2-7B creative finetune #2** | 7B creative/roleplay finetune | **92%** (11/12, parser-reject) | — | ~0% (15/15) | **needs grammar / avoid native tools** |

† `required` reads 0% for the fragile-parser model only because at the reduced scale it wrote anyway — it
does **not** prove enforcement. An adversarial test (prompt: "tell a joke, do NOT write") shows
`tool_choice: required` is **ignored** by that parser. For the Hermes-family models, the 0% under
`required` **is** genuine enforcement (the parser backs it with a grammar).

Bigger models of each family (a ~26B and ~31B fragile-family model, and a ~35B Hermes-family MoE) were
out of memory-budget for this sweep and are deferred; the parser-family prediction is that size does not
change the verdict — the fragile-family bigs still need the grammar, the Hermes-family MoE is still fine
natively — but **that must be measured, not assumed.**

### Load-dependence (an important nuance)

The fragile-parser model was **clean at reduced prompt scale** (short artifact) yet flaked ~25% at
**full writers-room scale** (a long, multi-section artifact). Narration emerges under heavier generation
load — a longer artifact gives the model more room to drift into "…has been written to `X`." So: a model
that looks fine on a toy prompt can still flake in production. **Measure at your real artifact size.** A
weaker model (the 3B) flakes even at the small scale; a broken finetune fails regardless of scale.

Served identically: one llama.cpp Metal build, `--jinja` on, reasoning/thinking off for tool turns,
each model on its own server instance, same brief + system prompt, same faithful 6-tool read→write loop.

## Reading the matrix — by parser family

- **Hermes-style native parser (Qwen3 family).** `<tool_call>…</tool_call>` with a strict, grammar-
  backed parser. `tool_choice: required` is genuinely *enforced* here (constrained decoding under the
  hood), and even `auto` is reliable on the real loop. **Verdict: works natively — do not pay the
  grammar tax.** This held across sizes in the family we tested.
- **Fragile family parser (Gemma-4).** A newer, dedicated parser that treats `tool_choice` as advisory
  and narrates under agentic load. `required` does *not* fix it; **grammar does. Verdict: needs grammar.**
- **Llama-3.2 / other-lineage finetunes.** Tool reliability is inherited from the *base lineage's*
  parser + how much the finetune preserved (or eroded) tool-use behavior — creative/roleplay finetunes
  often trade tool discipline for prose fluency. **Verdict: must be measured per-checkpoint** — see the
  numbers. A finetune that flakes gets the grammar path; the grammar (sampler-level) does not care that
  the weights were re-tuned.
- **Small MoE / other small dense.** Active-param count is small, so a priori suspect; measured result
  in the table.

## Practitioner takeaways

1. **Gate the grammar on measured need, not on "it's small."** A native-reliable family (Qwen3-style)
   should keep native tools — grammar adds a (small) constraint-tax risk and bypasses the model's real
   multi-tool reasoning for no benefit. Reserve the grammar path for the families that actually narrate.
2. **`tool_choice: required` is a per-family lever, not a portable one** (see [§2 of the main writeup](../README.md#2-what-does-not-work-tool_choice-required)).
   It's a real fix *only* where the parser enforces it (Qwen3-style). Where it's enforced you often don't
   need the grammar; where you need the grammar it isn't enforced. They are two tools for two regimes.
3. **Grammar is the universal floor.** For any model that flakes natively — fragile-parser family,
   eroded finetune, or a backend where the templated-tool path is broken — a loose GBNF grammar takes it
   to ~0 by construction. It is the one lever that works regardless of family or backend.
4. **Beyond GBNF, per model:** a native-reliable model wants *native tools* (best quality, real tool
   loop); a fragile-parser model wants *grammar* (single-output phases) or *routing to a
   native-reliable model* (genuine multi-tool loops); a persistently-bad checkpoint is a candidate for a
   *tool-use fine-tune* or simply *not being used for tool phases*. Retry-with-feedback sits behind all
   of these as the residual net, never as the primary fix. See the
   [landscape doc](./methodology-landscape.md).
