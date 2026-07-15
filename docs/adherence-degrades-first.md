# Adherence degrades first: why your model stopped listening before it stopped working

There is a pattern hiding across the quantization, abliteration, and creative-finetuning
literature that nobody states plainly, because each field measures it separately:

> **Instruction adherence is a *behavioral gate*, and it is the first thing to break.
> Knowledge and prose survive it. And the metrics everyone watches — perplexity, MMLU,
> accuracy — are nearly blind to it.**

Every lever that makes a model cheaper, freer, or more creative buys that gain **partly
with obedience**. If your pipeline depends on the model *doing what it was told* — write
this file, honor this note, stop now — that is the capability you are spending, and your
dashboards won't show it.

> Labels: `[PEER-REVIEWED]` · `[BENCHMARK-MEASURED]` · `[COMMUNITY-ANECDOTE]` · `[MEASURED-RAW]` (we ran it).

## 1. Quantization eats the gate, not the knowledge

**Epistemic abstention collapses while perplexity barely moves.** `[BENCHMARK-MEASURED]`
[arXiv:2605.15208](https://arxiv.org/html/2605.15208v1) (IEEE Cloud Summit 2026), BBQ,
12,148 items, 911,100 inference records:

| | BF16 | 3-bit |
|---|---|---|
| **Unknown Selection Rate** (willingness to answer *"unknown"*) | 0.764 | **0.631** (−17.4%) |
| Stereotype Reliance | 0.175 | 0.242 (+37.8%) |

Their own framing: compression *"selectively degrades **epistemic calibration** — the
model's learned ability to recognize insufficient information and withhold judgment."*

**The disparity is the story.** Same paper: Mistral-7B at 3-bit shows a **10.2% perplexity
increase** against **17.7% new bias emergence** — a **173× gap** between what your metric
sees and what the behavior does.

**Mean confidence is a decoy.** `[PEER-REVIEWED]` Proskurina et al.,
[Findings of NAACL 2024](https://aclanthology.org/2024.findings-naacl.124/): under 4-bit
GPTQ, aggregate confidence is unchanged (Mistral **96.85 → 96.89**) while
**confidence in the *true class* drops significantly in every model tested**. Degradation
concentrates exactly where the model was already unsure.

**The clearest statement of the mechanism.** `[PEER-REVIEWED]` Fu et al.,
[EMNLP 2025](https://aclanthology.org/2025.emnlp-main.1548/): layer-wise probing shows
quantized models **retain internally truthful representations even while emitting false
outputs**. *The knowledge survives. The behavioral gating does not.*

**Where the cliff is:** ≥4-bit is broadly calibration-safe (converged across
[Decoding Compressed Trust](https://arxiv.org/abs/2403.15447) (ICML 2024),
[Quantization Hurts Reasoning?](https://arxiv.org/abs/2504.04823) (COLM 2025), and the
above). **3-bit is the cliff.** 2-bit breaks behavior, and *method* matters more than
bit-width there (vector quantization survives where scalar doesn't).

## 2. Abliteration eats the gate too

Abliteration ("refusal-direction ablation") projects out a single linear direction
mediating refusal — surgical, no retraining
(`[PEER-REVIEWED]` [Arditi et al., NeurIPS 2024](https://proceedings.neurips.cc/paper_files/paper/2024/file/f545448535dfde4f9786555403ab7c49-Paper-Conference.pdf)).
In principle it touches one behavior. In practice:

`[BENCHMARK-MEASURED]` [arXiv:2512.13655](https://arxiv.org/pdf/2512.13655) — 4 abliteration
tools × 16 models: **MMLU/HellaSwag mostly <2%**, but **IFEval (instruction-following)
degrades 2–8%**, and GSM8K is the most fragile (Heretic averaging **−7.81pp**).

Same shape: **knowledge ~intact, instruction-following measurably worse.** The damage
does not stay inside "refusals" — it leaks into general adherence. Note the spread is
tool-dependent; "uncensored = dumber" is *partly* measured, partly folklore, and the
answer is not one number.

## 3. Creative finetuning trades adherence for willingness

`[COMMUNITY-ANECDOTE — single source, unreplicated]` A community RP benchmark (Sonnet-judged
craft, 40 models) placed **all 7 uncensored/RP finetunes at ranks #34–40 of 40** — ~2 Likert
points below the frontier cluster — with judges penalizing repetition, purple prose, and
agency violations. The one axis where they clearly win: **willingness** (0% refusal vs 5–10%).

They trade **craft and adherence for compliance**. Treat the ranking as directional (one
judge, one repo), but it converges with every creative-writing leaderboard, where base and
frontier models top the boards and no finetune does.

## 4. The failure has a name, a number, and a warning

`[PEER-REVIEWED]` **CODESTRUCT**, [arXiv:2604.05407](https://arxiv.org/abs/2604.05407)
(ACL 2026), SWE-bench Verified. An **empty patch** is
*"a failure to externalize an intended edit rather than incorrect high-level reasoning."*

| Model | Baseline → Structured | Empty patches |
|---|---|---|
| **GPT-5-nano** | 19.6% → **40.4%** (+20.8 pp) | **46.6% → 7.2%** |
| GPT-5 | 66.0% → 67.2% | — |
| **Qwen3-8B** | 13.2% → **13.0%** (−0.2 pp) | recovered 41, **gained zero** |

That is the peer-reviewed name for *narrate-don't-write*: **the model knew, and failed to
emit.** At 46.6% for a small model it is the dominant failure mode — and restructuring the
output channel takes it to 7.2%.

**The Qwen3-8B row is the warning.** Fixing the channel recovers work *only if the content
underneath was good*: 41 patches recovered, **zero** accuracy gained.

> **Calibrate: better channels and output-aware success recover work you were throwing
> away. They do not make the model smarter.** Still worth having — silent loss is worse
> than visible mediocrity — but don't sell it as a quality win.

## 5. What follows: buy adherence from the pipeline, not the model

If adherence is the first thing to degrade and your metrics can't see it, then hardening the
*model* is the wrong lever. Spend the model's entire budget on the thing that survives —
prose — and get obedience from code:

1. **Make the instruction legible.** Anchor it to the exact text it refers to. "Change this
   name" is unresolvable; *"change `Mei` at episode 1, passage → wants: a Latino name"* is a
   checkable contract.
2. **Take effects away from the model.** It emits text; deterministic code writes the file.
   It cannot fail to externalize an edit it was never asked to externalize. See
   [capture vs. tools](./capture-vs-tools-workload-shape.md).
3. **Verify deterministically — never with a judge.** LLM judges score **~AUROC 0.65** on
   *"did the state actually change"* ([arXiv:2606.09863](https://arxiv.org/abs/2606.09863))
   because they read confident closing language rather than verified state. Deterministic
   gates read the state, and lifted pass@5 consistency 8%→26%
   ([arXiv:2607.07405](https://arxiv.org/html/2607.07405v1)). A string check over
   before/after, keyed on the instruction's anchor, answers *honored / ignored /
   unverifiable* — and **`unverifiable` must be a first-class verdict**, or an over-eager
   check recreates the problem it exists to catch.
4. **Re-ask by naming the specific defect.** Not a retry — *"you left `Mei` unchanged; the
   note asked for a Latino name."* Blind retry is useless-to-harmful: greedy decoding
   reproduces the same output and failed attempts poison the context. Bound it (~3), then
   **escalate** rather than loop.
5. **Don't route adherence-critical work to the models that stopped listening.** IQ2,
   abliterated, and creative-finetuned models are precisely the ones whose gate is gone.

**The one-line version:** *every lever that makes a model cheaper or more creative makes it
listen less — and perplexity won't tell you. So stop buying obedience from the model. Make
the instruction checkable, take the effects away, verify with code, and re-ask by name.*
