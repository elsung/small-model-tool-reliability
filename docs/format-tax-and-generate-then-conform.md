# Does constraining output format hurt quality? The format-tax controversy, resolved

Short answer, as of 2026: **the token mask is cheap; the prompt is expensive.** The widely-cited
claim that constrained decoding damages reasoning attributes to the *decoder* an effect that is
mostly caused by the *format-requesting instructions in the prompt* — and it is largely an
**open-weight** phenomenon. The configuration that costs you nothing is **generate free, then
conform**.

This matters because the original "constraints hurt reasoning" headline is still repeated as
settled fact, and it argues against the exact lever ([grammar-constrained decoding](../README.md))
that fixes narration and non-termination.

> **Provenance labels used throughout.** `[PEER-REVIEWED]` · `[PREPRINT]` (no venue, treat as a
> hypothesis) · `[VENDOR]` (self-run, unaudited, grading their own product) · `[MEASURED-RAW]`
> (we pulled the primary data). The honest story here is that **the vendor numbers on both sides
> sit inside their own noise bands** — which is why the labels are not decoration.

## 1. The claim that started it

**"Let Me Speak Freely? A Study on the Impact of Format Restrictions on LLM Performance"** —
Tam et al. (Appier), EMNLP 2024 Industry Track. `[PEER-REVIEWED]`
[arXiv:2408.02442](https://arxiv.org/abs/2408.02442) · [ACL](https://aclanthology.org/2024.emnlp-industry.91.pdf)

> "Surprisingly, we observe a significant decline in LLMs reasoning abilities under format
> restrictions. Furthermore, we find that stricter format constraints generally lead to greater
> performance degradation in reasoning tasks."

GSM8K, Text vs `format + schema` (std in parens):

| Model | Text | JSON+schema | XML+schema | YAML+schema |
|---|---|---|---|---|
| Gemini-1.5-Flash | 89.33 (0.8) | 89.21 (1.5) | 88.20 (2.2) | 87.42 (3.7) |
| Claude-3-Haiku | 86.51 (0.8) | **23.44 (22.9)** | 79.76 (7.0) | 80.63 (2.8) |
| GPT-3.5-Turbo | 75.99 (3.1) | 49.25 (12.0) | 45.06 (19.9) | 73.85 (5.6) |
| LLaMA-3-8B | 75.13 (0.9) | 48.90 (6.7) | 56.74 (8.3) | 46.08 (16.8) |

## 2. Why the headline does not survive scrutiny

**The variances are the story.** Claude-3-Haiku JSON GSM8K = **23.44 with std 22.9**; GPT-3.5 XML =
45.06 with **std 19.9**. When the standard deviation approaches the mean, you are measuring
*prompt sensitivity*, not capability.

**Its own results contradict its headline.** Table 10 shows format restriction *helping*
classification: Gemini DDXPlus **41.6 → 60.3**; Gemma2-9B DDXPlus **22.9 → 53.0**; GPT-3.5 Sports
67.2 → 80.0. The paper concedes constraints "hinder reasoning abilities while enhancing
classification task accuracy."

**It conflates JSON-mode with structured generation.** JSON-mode ("valid JSON, no schema") and
grammar-constrained generation against a real schema are different mechanisms with different
failure modes. Much of the measured damage belongs to the former.

## 3. The vendor rebuttal — right diagnosis, unproven claim

**dottxt, "Say What You Mean"** `[VENDOR]`
[blog.dottxt.ai/say-what-you-mean.html](https://blog.dottxt.ai/say-what-you-mean.html)

Their *methodological* critique is strong and independently accepted: the NL and structured prompts
were **not apples-to-apples**, and "**the paper confuses structured generation with JSON-mode**."

Their *empirical* claim ("structured generation is an improvement across the board") is **not
established**: Llama-3-8B GSM8K 0.77 → 0.78, Last Letter 0.73 → 0.77 — deltas of **+0.01 to +0.04**,
inside the very noise band their own critique says invalidated Tam et al. Self-chosen prompts, no
preregistration, vendor grading its own product.

## 4. Independent confirmation of the confound

**JSONSchemaBench** — Geng et al. `[PEER-REVIEWED, CONFLICTED]` (authored by the guidance-ai/Microsoft
team; Guidance wins nearly every metric it reports)
[arXiv:2501.10868](https://arxiv.org/html/2501.10868v1) · [repo](https://github.com/guidance-ai/jsonschemabench)

They acted on the critique — *"For the task of Last Letter, we make a slight modification because
**the original prompt used was a bad example** as pointed out by Kurt (2024b)"* — and found the
opposite of Tam's headline:

> "constrained decoding, regardless of the framework, achieves higher performance than the
> unconstrained setting" (~3%)

GSM8K: LM-only **80.1** · Outlines 81.6 · **llama.cpp 82.4** · XGrammar 83.7 · Guidance 83.8.

## 5. The resolution (2026): the cost is in the prompt, not the decoder

**"The Format Tax"** — Lee, D'Antoni, Berg-Kirkpatrick. `[PEER-REVIEWED]`
[arXiv:2604.03616](https://arxiv.org/abs/2604.03616) — 6 open-weight + 4 API models.

> "sampling bias accounts for only a fraction of the degradation. **The dominant cost enters at the
> prompt: format-requesting instructions alone cause most of the accuracy loss, before any decoder
> constraint is applied.** ... decouple reasoning from formatting ... **Notably, most recent
> closed-weight models show little to no format tax**"

Two consequences that reorganize the whole debate:

1. **The token mask is not the problem.** Piling "you MUST output exactly…" into the prompt is.
2. **The format tax is an open-weight phenomenon.** It hits locally-served models and largely spares
   recent closed-weight APIs — so "does constraining hurt?" has *different answers for your local
   fleet and your cloud tier*. Most of the argument has been people generalizing from one to the other.

**"Thinking Before Constraining"** `[PREPRINT]` [arXiv:2601.07525](https://arxiv.org/abs/2601.07525) —
constraints "imposing constraints too early in the generation process" truncate reasoning; reason
free, constrain after a trigger token → "virtually eradicat[es] premature triggering", with
"accuracy gains of up to 27% over natural generation."

## 6. The convergence — three papers, three domains, one prescription

The most-corroborated finding in this entire body of work. Three independent 2026 results, from
unrelated domains, prescribe the same pipeline shape:

| Paper | Domain | Prescription |
|---|---|---|
| **The Format Tax** ([2604.03616](https://arxiv.org/abs/2604.03616)) | reasoning + writing | **2-Turn**: reason free → then format (**+6.8 pp**) |
| **Dramaturge** | screenplay revision | **Phase gate**: structure pass → lock storyline → then prose |
| **Constraint Tax** ([2606.25605](https://arxiv.org/abs/2606.25605)) | tool calling | **Two-Pass**: decouple execution from schema-constrained generation |

> **Separate the generation pass from the conformance pass.** Generate freely; conform afterward
> (a second turn, a lazy grammar triggered at the structural anchor, or a deterministic extractor).
> Never clamp the constraint on at token zero.

## 7. ⚠️ Tool Suppression — grammar and tools can be mutually exclusive

**"Constraint Tax"** `[PREPRINT — no venue]` [arXiv:2606.25605](https://arxiv.org/abs/2606.25605).
Mechanism verified from the abstract:

> "when Tool Calling and JSON Schema constraints are simultaneously enabled, multiple open-weight
> models cease invoking tools despite maintaining high schema compliance" — because "JSON Schema
> constraints are compiled into grammar-based token masks, **causing tool-call tokens to become
> unreachable during decoding**."

**The numbers are NOT in the abstract and are unverified — do not cite figures from this paper.**
The *mechanism*, however, is a concrete hazard for anyone combining a grammar with tool-calling, and
a plausible root cause of the **"writes the file fine but never terminates"** failure: if the
terminal tool-call token is masked unreachable, the model **cannot** emit it.

**Test this before building a terminal-grammar fix on top of an active tool loop.** Either drop the
tools (capture mode — see [capture vs. tools](./capture-vs-tools-workload-shape.md)) or verify your
terminal token is still reachable under the mask. The two levers can silently cancel each other.

## 8. Practitioner takeaways

1. **Stop citing "constraints hurt reasoning" as settled.** The primary source has std devs up to
   22.9, contradicts itself on classification, and conflates JSON-mode with structured generation.
2. **Budget the tax where it actually is: the prompt.** Minimal format instructions beat elaborate
   "you MUST" preambles. This is the single cheapest win available.
3. **Constrain late, never at token zero.** Lazy/triggered grammars, or a second conformance turn.
4. **Loose grammar for prose, strict for data.** Anchor the *skeleton* (a heading, a terminal
   object); leave sentences free. Over-tight grammars on creative text spend quality for nothing.
5. **Ask "open or closed weights?" first.** The format tax largely spares recent closed models. A
   result measured on one tier does not transfer to the other.
6. **Never combine a grammar mask with an active tool loop without testing terminal-token
   reachability** (§7).
7. **Engine choice:** llama.cpp GBNF benchmarks well (82.4 GSM8K, 0.20s TTFT). Outlines carries a
   **3.48s grammar compile** and the weakest schema coverage in JSONSchemaBench (0 of 13 test-suite
   categories at full coverage) — price that against your latency budget, noting the measurement
   comes from a competing vendor.

### A note on epistemic hygiene

An automated extraction pass on the Tam et al. paper **fabricated a results table** (a clean
"NL / JSON-Mode / FRI" grid) that does not exist in the paper — its real appendix columns are
Text/JSON/XML/YAML. Every figure here was re-fetched under a "printed cells only" rule. If you see
that NL/JSON-Mode/FRI grid attributed to this paper anywhere, it is invented. Extraction agents
interpolate; verify before you cite.
