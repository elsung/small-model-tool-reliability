# The broader landscape: constrained decoding & tool-use reliability

The [main writeup](../README.md) argues that **grammar-constrained decoding** is the durable fix for small-model tool/structured-output reliability. This document places that technique in its wider context: the engines that implement it, the tradeoffs (including the genuine, *contested* "constraint tax"), the academic grounding, and the complementary approaches (fine-tunes, parsers, agent patterns). The goal is to explore *beyond* GBNF — this is a survey, not a pitch.

> Citations are to official docs, project repos, and arXiv/ACL. Where a specific quantified claim is cited, treat it as "as reported by that source" — reproduce before relying on it. A few IDs are flagged where they could not be fully confirmed.

---

## A. The mechanism, in one paragraph

All of these techniques do the same core thing: at each decoding step, compute the set of tokens that keep the output within a formal specification (a regex, a JSON Schema, or a context-free grammar), **mask every other token's logit to `-inf`, then sample normally.** The output is guaranteed to parse. They differ in (1) what specification language they accept, (2) how fast they compute the per-step token mask, and (3) whether they correct the distribution distortion that masking introduces. This is variously called *constrained decoding*, *guided decoding*, *structured generation*, or *structured outputs*.

---

## B. The engines

### llama.cpp — GBNF + LLGuidance
- **GBNF** (GGML BNF) is llama.cpp's native grammar format — BNF extended with regex-like character ranges and repetitions; the output shape is defined by the `root` rule and applied by the sampler. A documented performance gotcha: use `x{0,N}` rather than repeated `x? x? x?…`, which causes "extremely slow sampling."
  - Grammar guide: <https://github.com/ggml-org/llama.cpp/blob/master/grammars/README.md> · JSON example: `grammars/json.gbnf`
- **`--grammar` / `--grammar-file`** — CLI flags for grammar-constrained sampling; in the server, passed as the `grammar` body field. Server README: <https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md>
- **`response_format: json_schema`** — converts a subset of JSON Schema (≈Draft 7) to GBNF; `/v1/chat/completions` supports `json_object`, `json_object` + `schema`, and `json_schema`. Note from the docs: *"the JSON schema is only used to constrain the model output and is not injected into the prompt."* Converter: `examples/json_schema_to_grammar.py`.
- **Lazy grammars** (`grammar_lazy` / `grammar_triggers`) activate the constraint only after a trigger is emitted (e.g. after `</think>`) — "free reasoning, then forced structure." *Caveat:* this is exposed on the `/completion` route, **not** on the OpenAI-compatible `/v1/chat/completions` route.
- **LLGuidance** — a fast Rust constrained-decoding backend (guidance-ai, originally Microsoft) integrated into llama.cpp (build `-DLLAMA_LLGUIDANCE=ON`); supports JSON Schema + arbitrary CFGs. Reported token-mask cost ≈ **50µs single-core** (p99 0.5ms) on the Llama-3 128k-vocab tokenizer. Docs: <https://github.com/ggml-org/llama.cpp/blob/master/docs/llguidance.md> · <https://github.com/guidance-ai/llguidance>

### vLLM — structured outputs / guided decoding
- Request params `guided_json` / `guided_grammar` (EBNF CFG) / `guided_regex` / `guided_choice`, plus `structural_tag`. **Version note:** as of **v0.12.0** these are deprecated in favor of a single `structured_outputs` param, and the per-request `guided_decoding_backend` field was removed — cite the version you target.
- Backends: **XGrammar** (primary under the default `auto`), **guidance/llguidance**, **outlines**, **lm-format-enforcer**. Docs: <https://docs.vllm.ai/en/latest/features/structured_outputs.html>

### SGLang — XGrammar + compressed FSM / jump-forward
- Paper: *"SGLang: Efficient Execution of Structured Language Model Programs,"* Zheng et al., NeurIPS 2024, `arXiv:2312.07104`. Combines **RadixAttention** (automatic KV-cache reuse via a radix tree) with a **compressed finite-state machine** for constrained decoding; reports up to **6.4× throughput** vs prior systems.
- **Jump-forward decoding**: when the grammar forces a deterministic multi-token span (e.g. the literal `"name":`), it is emitted in one step instead of sampled token-by-token, cutting overhead on heavily-templated output. Blog: <https://lmsys.org/blog/2024-02-05-compressed-fsm/>
- Backends via `--grammar-backend`: XGrammar (default), Outlines, LLGuidance. Docs: <https://docs.sglang.io/advanced_features/structured_outputs.html>

### XGrammar
- A high-performance structured-generation engine (MLC/CMU) for general CFGs. Speed comes from splitting the vocabulary into **context-independent tokens** (>99% of the mask, precomputable) vs context-dependent, an **adaptive token-mask cache**, a pushdown automaton, and **overlapping CPU mask computation with GPU inference**.
- Reported: grammar-engine speedup **up to 3.5× (JSON) / 10× (CFG)** vs Outlines/llama.cpp/lm-format-enforcer; headline **"up to 100×"** end-to-end with near-zero overhead. Now the default structured-output backend in vLLM, SGLang, TensorRT-LLM, MLC-LLM.
- Paper `arXiv:2411.15100`; repo <https://github.com/mlc-ai/xgrammar>; blog <https://blog.mlc.ai/2024/11/22/achieving-efficient-flexible-portable-structured-generation-with-xgrammar>.

### Outlines
- Regex- and JSON-Schema-guided generation reformulated as transitions between **FSM states**, with an **index over the vocabulary** so the allowed next-token set is a lookup, not a scan → claimed **O(1) per-token** overhead, model-agnostic.
- Paper: *"Efficient Guided Generation for Large Language Models,"* Willard & Louf, `arXiv:2307.09702`; repo <https://github.com/dottxt-ai/outlines>.

### lm-format-enforcer
- Enforces JSON Schema / regex by **filtering allowed tokens each step** (intersecting a character-level parser with a tokenizer prefix tree). Distinguishing design: it lets the **model keep control over field ordering and whitespace**, and supports **batching and beam search**. Differentiator is flexibility, not raw speed. Repo <https://github.com/noamgat/lm-format-enforcer>.

### Microsoft / guidance
- A DSL that **interleaves control flow with generation**, constraining via regex, CFG, `select()`, and JSON schema. Features **token healing** (removes tokenization-boundary bias) and **guidance acceleration** (deterministically-fixed tokens inserted without a forward pass, KV-cache reuse). Now maintained by `guidance-ai`; its fast backend is llguidance. Repo <https://github.com/guidance-ai/guidance>.

### Provider "structured outputs" (JSON-Schema strict mode)
- Hosted APIs productize the same mechanism: **OpenAI Structured Outputs** (`response_format: json_schema`, `strict: true`) compiles the schema to a grammar and masks violating tokens server-side, guaranteeing schema-conformant JSON. Reported on OpenAI's complex-schema eval: `gpt-4o-2024-08-06` with Structured Outputs **100%** vs prompting-only **<40%**. Announcement: <https://openai.com/index/introducing-structured-outputs-in-the-api/>. (Anthropic and Google offer analogous JSON/schema modes.) This is strong evidence that constrained decoding is the industry-standard route to reliable machine-readable output.

---

## C. The tradeoffs

### The "constraint tax" — and the debate about it
The central caution, and a genuinely *contested* one worth presenting honestly:

- **The concern:** *"Let Me Speak Freely? A Study on the Impact of Format Restrictions on Performance of Large Language Models,"* Tam et al., `arXiv:2408.02442` (EMNLP 2024 Industry). Finds that strict formats (especially JSON-mode, which forces answer ordering and can crowd out chain-of-thought) **degrade reasoning**; looser format instructions degrade less. Reported GSM8K drops NL→JSON-mode: GPT-3.5-Turbo 76.6%→49.3%, Claude-3-Haiku 86.5%→23.4%, LLaMA-3-8B 74.7%→48.9%.
- **The rebuttal:** dottxt, *"Say What You Mean: A Response to 'Let Me Speak Freely',"* argues the study conflated no-guarantee "JSON-mode" with true constrained generation and used mismatched prompts; with matched prompts, structured generation **matched or beat** unstructured (e.g. GSM8K 0.77→0.78). <https://blog.dottxt.ai/say-what-you-mean.html>. dottxt separately reports structure *helping* GSM8K on Mistral-7B (+17.5% from JSON-formatted prompt, +20.7% when enforced): <https://blog.dottxt.ai/performance-gsm8k.html>.
- **The mechanistic middle:** the disagreement is largely explained by *how* you constrain (see GAD below) and *how tight* the grammar is. **The operational rule used in the field report: keep grammars loose.** Force only the structural skeleton the parser needs (opening heading, section markers, required JSON keys) and leave the body permissive. Empirically that took narration 25% → 0% with quality preserved. A recent size-dependent study — *"The Constraint Tax: Measuring Validity–Correctness Tradeoffs…for Small Language Models,"* `arXiv:2605.26128` — reports that constraints reliably raise *validity* but can cut *correctness*, more so on smaller models (specific figures not independently verified here).

### Distribution distortion — masking ≠ the model's true conditional
Masking illegal tokens and renormalizing does **not** reproduce the model's true distribution conditioned on "the output is grammatical"; greedy per-token masking can steer toward locally-legal but globally-unlikely continuations.
- **Grammar-Aligned Decoding**, Park et al., `arXiv:2405.21047` (NeurIPS 2024) — formalizes this and gives **ASAp** (Adaptive Sampling with Approximate expected futures), which provably converges to the grammar-conditioned model distribution and yields higher-likelihood outputs. A follow-up, *Constrained Adaptive Rejection Sampling (CARS)*, `arXiv:2510.01902`, approximates the same via adaptive rejection sampling *(author list unverified)*.

### Speed & expressiveness
- Naive per-step grammar checking is expensive; XGrammar's mask cache, Outlines' FSM index, and SGLang's jump-forward exist precisely to drive that overhead toward zero. With a modern engine, constrained decoding is close to free; with a naive one it can dominate latency.
- **Expressiveness ladder:** regex < JSON-Schema < full CFG. Some backends cover only a subset. Most "produce this document / this JSON" needs are met by JSON-Schema or a shallow CFG — reach for a full CFG only when the artifact is recursively structured.

---

## D. Academic grounding (quick index)

| work | arXiv | contribution |
|---|---|---|
| Grammar-Constrained Decoding without Finetuning (Geng et al., EMNLP 2023) | `2305.13971` | reference GCD formulation; input-dependent grammars; unfinetuned LLMs match finetuned on structured tasks |
| Efficient Guided Generation (Willard & Louf, 2023) | `2307.09702` | FSM vocabulary indexing → O(1)/token; the Outlines method |
| SGLang (Zheng et al., NeurIPS 2024) | `2312.07104` | RadixAttention + compressed-FSM constrained decoding |
| XGrammar (Dong et al., 2024) | `2411.15100` | efficient CFG engine now embedded across serving stacks |
| Grammar-Aligned Decoding (Park et al., NeurIPS 2024) | `2405.21047` | fixes GCD's distribution distortion (ASAp) |
| Let Me Speak Freely? (Tam et al., EMNLP 2024) | `2408.02442` | the constraint-tax evidence (contested — see rebuttal above) |

---

## E. Complements & alternatives (for tool use specifically)

Constrained decoding guarantees *shape*. It does not teach a model *when* to call a tool or *which* one. These address the rest and **compose** with it.

### Tool-use fine-tunes (behavior baked into weights)
- **NousResearch Hermes function-calling** — ChatML + dedicated `<tool_call>` / `<tool_response>` tokens and a `tool` role; the "Hermes-style native parser" referenced throughout this report, and a big reason a Qwen3-14B with a Hermes-style parser is *enforceable* where a fragile-parser model is not. Reported (model card): **90% function-calling eval, 84% structured-JSON eval**. <https://github.com/NousResearch/Hermes-Function-Calling>
- **Gorilla / gorilla-openfunctions** (Berkeley), `arXiv:2305.15334` — LLaMA tuned to emit correct API calls, retriever-augmented for changing docs; introduced APIBench. <https://github.com/ShishirPatil/gorilla>
- **ToolLLM / ToolBench** (Qin et al.), `arXiv:2307.16789` — 16k+ real REST APIs, ToolLLaMA, DFSDT, ToolEval. <https://github.com/OpenBMB/ToolBench>

### Better serving parsers & serving hygiene
Often the cheapest win is a *correct* serve, not a new model: enable template rendering, pick a model family with a mature tool parser, disable thinking on tool-critical turns, keep the build current, avoid aggressive KV-quant (see the [hardware checklist](./hardware-notes.md)). A healthy native path plus `tool_choice: required` *is* real constrained decoding — **when the parser backs it** (the whole point of the main writeup's §2).

### Agent-framework patterns
- **ReAct**, Yao et al., `arXiv:2210.03629` — interleave reasoning traces with actions; the archetypal tool loop (+34% abs. on ALFWorld vs baselines, as reported).
- **Format-then-parse** — emit a constrained intermediate (JSON / grammar-forced artifact), then parse & validate in code rather than trusting a native tool channel. The grammar path in the main writeup is a hardened version of this.
- **Retry-with-feedback / self-correction** — e.g. **Reflexion** (Shinn et al., `arXiv:2303.11366`): on a missing/malformed call, re-inject the error and re-request. It's reactive (lets the failure happen, then cleans up), so it belongs *behind* constrained decoding as the residual net, not instead of it.

### Benchmarks (measure, don't assume)
- **Berkeley Function-Calling Leaderboard (BFCL)** — AST accuracy, executable calls, irrelevance/hallucination detection, parallel/multiple calls; **V3 adds multi-turn/multi-step** with state-based verification. <https://gorilla.cs.berkeley.edu/leaderboard.html>
- **RoTBench**, Ye et al., `arXiv:2401.08326` — tool-use *robustness* under five noise levels × three phases (selection / parameter-id / content-filling).
- Constrained decoding tends to move the **format-validity** axis to ~100%; these benchmarks check whether **correct-tool / correct-args** also improved — validity is necessary, not sufficient.

---

## F. Quantified: does constraining actually *lift* small-model reliability?

Yes on validity, and often on correctness — with the largest gains on the smallest models (which is exactly where the field report's problem lives). Representative reported numbers (reproduce before relying):

- **XGrammar-2 on BFCL-v3** (`arXiv:2601.04426`; a Jan-2026 follow-up — *existence verified, treat as a distinct paper*): schema validity → **100%** across sizes, with correctness gains largest on small models:

  | model | correct-call (off → on) | schema validity (off → on) |
  |---|---|---|
  | Llama-3.2-1B | 6.1% → 32.8% | 22.1% → 100% |
  | Llama-3.2-3B | 33.1% → 77.8% | 40.7% → 100% |
  | Llama-3.1-8B | 59.5% → 80.9% | 67.0% → 100% |
  | Llama-3.1-70B | 45.6% → 86.4% | 51.9% → 100% |

- **Guided decoding on vLLM & SGLang** (SqueezeBits): schema-following tasks lifted from ~60–90% to as high as **98–100%** with XGrammar/LLGuidance. <https://blog.squeezebits.com/guided-decoding-performance-vllm-sglang>
- **Provider structured outputs** (OpenAI): complex-schema conformance **<40% → 100%**.

The through-line matches the field report: **small models benefit the most from constrained decoding, and validity is the axis that snaps to ~100%.** The caution (constraint tax) is real but is mitigated by loose grammars and, in principle, by distribution-aligned decoding.

---

## G. Future directions & open questions

- **Constrained + speculative decoding.** The draft model must also respect the grammar; making the two compose cleanly would erase the last of the constraint's latency cost. Largely an open engineering frontier in open stacks.
- **Grammar-guided *multi-step* tool loops.** The field report constrains *single-output* phases. Extending token-level enforcement across a whole agent loop ("call tool → read result → call next tool"), not just the final artifact, is mostly unsolved openly.
- **Per-model grammar libraries.** Families tokenize structure differently; a reusable library of loose "artifact-shape" grammars (plus tooling to derive them from a JSON Schema) would make adoption turnkey.
- **Measuring quality-vs-constraint directly.** The constraint tax is under-quantified per task/model/grammar. A cheap protocol to A/B a phase with vs. without a grammar — and to auto-loosen to the minimum that still guarantees parse — would replace guessing with tuning.
- **Distribution-aligned decoding in production.** GAD/ASAp fixes the distortion in principle; getting its cost low enough for a latency-sensitive serving path is open.
- **A portable "forced call" contract.** Today `tool_choice: required` means "enforced" on one stack and "hint" on another. A cross-backend guarantee that forcing a call is *always* backed by constrained decoding would make the failure in this report impossible by construction.
