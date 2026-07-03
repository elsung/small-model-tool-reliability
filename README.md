# Making Small Models Reliable at Tool Calling

**Why small local models narrate tool calls instead of making them — and how grammar-constrained decoding fixes it.**

A field report + methodology survey on getting reliable **tool use** and **structured output** out of small, locally-served instruction-tuned models (Gemma-4-12B class, Qwen3-14B class) inside an agentic, multi-tool, multi-turn loop.

The short version: on a real multi-tool write phase, a Gemma-4-12B-class model **narrated completion** ("the outline has been written to `…`") instead of emitting the tool call about **25% of the time**. The obvious lever — `tool_choice: required` — does **not** fix it, because the serving stack does not grammar-enforce it for this model family. **Grammar-constrained decoding** does fix it: it masks illegal tokens *before sampling*, so narration is physically impossible. Measured result: **25% → 0% narration over ~45 real runs**, with output quality preserved.

---

## Contents

- [1. The problem](#1-the-problem)
- [2. What does *not* work: `tool_choice: required`](#2-what-does-not-work-tool_choice-required)
- [3. What works: grammar-constrained / guided decoding](#3-what-works-grammar-constrained--guided-decoding)
- [4. The broader landscape (beyond GBNF)](./docs/methodology-landscape.md)
- [5. Hardware & serving-backend notes](./docs/hardware-notes.md)
- [Appendix: example grammars](./examples/)

---

## 1. The problem

### The setup

Consider a document-generation pipeline built as an **agent loop**: an agent CLI that speaks the OpenAI `tools` API drives a locally-served model through phases like *"read the source brief, compose an outline, write it to `outline.md`."* Each phase gives the model a handful of tools — `read`, `find`, `grep`, `write`, `edit`, `bash` — and expects it to **read → compose → call `write`**. The turn is only "done" when a file actually appears on disk.

Frontier closed-API models (the large hosted tier) do this reliably. Small locally-served models do not.

### The failure

On a real outline phase — 6 tools, a two-step read-then-write loop, mid-size (20k+ token) prompt — a **Gemma-4-12B-QAT** model would frequently do this:

1. Call `read` on the source brief. Good.
2. Generate the full outline **as text in its reply**.
3. End with: *"The outline has been written to `…/outline.md`."*
4. …never call `write`.

The agent loop sees no tool call, considers the turn complete, exits `rc=0` — **and no file exists.** The model *hallucinated the completion*. Worse, this is silent: the run looks successful.

Measured on the real agent CLI with the real prompt: **4 / 16 trials narrated with no file written ≈ 25% flake.** (Note: many *successful* trials also emitted "the file has been written to X" text *alongside* the real `write` call — narration text is common and benign. The failure is narration **instead of** the call.)

### Why small models do this and big ones don't

Native "function calling" is not a primitive the model executes deterministically. It is the product of **two learned/engineered layers**, both weaker on small local models:

1. **The model's trained tool-use behavior.** Emitting a well-formed tool call at the right moment (instead of describing the action in prose) is a behavior instilled by post-training. Large frontier models have deep, robust tool-use training; a 12B instruction tune has a thinner, more fragile version of it. Under distribution shift — long context, multiple tools, a two-step loop, higher sampling temperature — the small model falls back to what it does best: **writing fluent prose about the task**, including prose that *asserts the task is done*.

2. **The serving stack's tool parser.** For native tool calling to work, the server must (a) render the prompt through the model's real chat template so the `tools` are actually injected, and (b) parse the model's output back into a structured `tool_calls` field. In llama.cpp this path is gated behind template rendering (the `--jinja` family of flags) and a **per-model-family parser**. The Gemma-4 parser is comparatively new and fragile (multiple upstream reports of tool calls being emitted as plain text, of repetition loops, and of format mismatches). When either layer degrades, the model's tool intent lands in `message.content` as narration — indistinguishable from "it just decided to talk."

So the failure is **structural**, not a bug in any one place: a weaker-trained behavior meeting a more fragile parser, under agentic load. That framing matters, because it tells you where the durable fix has to live — *below* both layers, at the sampler.

---

## 2. What does *not* work: `tool_choice: required`

The intuitive fix is to remove the model's option to "just talk": set `tool_choice: "required"` (or force a named function). The OpenAI-style contract says this *must* produce a tool call. On some stacks it is even implemented as real constrained decoding.

**On the tested mid-2026 llama.cpp build, it does not work for the Gemma-4 family.** `tool_choice` is treated as a *hint the model can ignore*, not an enforced constraint, because the Gemma-4 tool parser does not wire `tool_choice` into a decoding grammar.

We proved this **adversarially**. Prompt: *"Tell me a joke, do NOT write any file, just talk."* With `tool_choice: "required"`:

| server / model (parser) | `tool_choice: "required"` on adversarial "tell a joke, don't write" | forced named function `{name: write}` |
|---|---|---|
| **Gemma-4-12B-QAT** (Gemma-4 parser) | **not forced** — told the joke, no tool call | **not forced** — no call |
| **Qwen3-14B** (Hermes-style / native parser) | **forced** — emitted a `write` tool call | (varies) |

The contrast is the whole point. A model whose serving parser has a **strict, grammar-backed tool path** (Qwen3-14B with a Hermes-style `<tool_call>…</tool_call>` parser) *is* enforced by `tool_choice: required`. A model whose parser treats it as advisory (Gemma-4) is not. **The enforcement lives in the parser, not in the API field** — and parser maturity varies by model family.

**Also don't:** serve one model under another model's tool chat-template to "borrow" the stricter parser. We tried serving Gemma-4 under a Qwen/ChatML tool template to reach the enforceable parser — it was *still* not forced, **and** it garbled the output, because the ChatML control tokens are foreign to Gemma's tokenizer/training. Serving a model under a foreign tool template is a reliable way to corrupt output, not to fix enforcement.

**Takeaway:** `tool_choice: required` is a real lever **where the serving parser enforces it** (Qwen3-style native parsers) and a no-op where it doesn't (Gemma-4 on this build). It is not portable across model families or serving backends. You cannot assume it works — you have to verify it per model + per build, adversarially.

---

## 3. What works: grammar-constrained / guided decoding

**Constrain generation at the token-sampling level.** A grammar (GBNF, or a JSON Schema compiled to one) defines exactly which token sequences are legal. At each decoding step the sampler tests candidate tokens against the grammar and **masks every illegal token to `-inf` before sampling**. The model literally cannot emit a token the grammar forbids.

If the grammar's root says the output *must* start with `## Episode 1` (or must be a JSON object with a `path` and `content` field), then **prose narration is not a reachable state.** "The outline has been written to…" cannot be produced, because none of its tokens are grammar-legal at the root. The failure isn't discouraged — it is made **impossible by construction.**

### The result

Same Gemma-4-12B-QAT, same phase, same prompts, now with a per-phase GBNF grammar forcing the artifact's structural skeleton:

| config | mechanism | N | valid / file written | narration flake |
|---|---|---|---|---|
| **before** — agent loop, 6 tools, `tool_choice: auto` | native tool call (fragile parser) | 16 | 12 | **25% (4/16)** |
| `tool_choice: required` (injected) | hint, ignored by this model | — | ≈ baseline | ~25% |
| **after** — grammar-constrained (GBNF) | **forced at the sampler** | 20 | **20 (100%)** | **0% (by construction)** |

Extended across three single-output phases (outline, beatsheet, "bible" build), the grammar path produced **45/45 clean runs: 0 narration, 0 malformed, 0 error.** An adversarial check held too: given `response_format: json_schema {path, content}` and the prompt *"tell me a joke, do NOT write a file, just talk,"* the model emitted valid `{"path": …, "content": …}` JSON. It *wanted* to just talk; the grammar left it no legal way to.

### Keep the grammar *loose* — avoid the "constraint tax"

The critical design rule: **force only the structural skeleton, keep the body permissive.** There is a real, measured phenomenon — over-tight formatting constraints can *degrade* a small model's task quality (it spends capacity satisfying the grammar instead of reasoning; see the [landscape doc](./docs/methodology-landscape.md) for citations). The mitigation is a loose grammar:

```gbnf
# Force the episode-section skeleton; let the body be rich free prose.
root     ::= section+
section  ::= "## Episode " [0-9]+ "\n" bodyline+
bodyline ::= [^\n]+ "\n"
```

This forces every output to open at a real artifact heading (killing narration/preamble), while the `bodyline` rule permits arbitrary prose, tables, and markdown. In practice, output **quality is preserved** — outputs open cleanly at the heading with rich, coherent content — because the grammar constrains *shape*, not *substance*.

### Why this is the *universal* lever

Grammar sampling is applied by the **sampler**, independently of the chat-template / tool-parsing pipeline. Every mainstream llama.cpp build ships the grammar sampler. This has a big consequence:

> **Grammar-constrained decoding works even on serving backends where the native templated-tool path is broken or unavailable.**

On a backend where the `--jinja` templated-tool path segfaults or is disabled (so native tool calling and `tool_choice` are effectively off), the grammar path **still works**, because it never touches that path. It is a per-request parameter, needs no serving-flag change, and turns a "capture-only" box into one that can produce forced-structured output. See [hardware notes](./docs/hardware-notes.md).

### Where it fits: reframing "capture mode"

If your pipeline already has a fallback that *parses freeform text* when a tool call is missing ("capture mode"), grammar-constrained decoding **upgrades and replaces it** for single-output phases:

- **Primary path (single-output phases):** call the server directly with the phase's grammar + a bounded `max_tokens`, then write the returned artifact to disk. This *skips the fragile tool loop entirely* — guaranteed structured, bounded, fast, ~0 narration.
- **Alternative for genuine multi-tool loops:** route to a model whose parser *does* enforce `tool_choice: required` (Qwen3-style).
- **Residual fallback:** best-effort parse-capture, now demoted to a rare-to-never safety net.

One practical caveat: with an open-ended grammar (`section+`), a large default `max_tokens` can let generation run away. **Cap `max_tokens` per phase** — it is required, not optional.

---

## 4. The broader landscape (beyond GBNF)

Constrained decoding is one family in a larger toolbox — vLLM/SGLang guided decoding, XGrammar, Outlines, lm-format-enforcer, Microsoft `guidance`, JSON-schema structured outputs — with real tradeoffs (the constraint tax, grammar expressiveness, throughput), and complementary approaches (tool-use fine-tunes, better parsers, ReAct/format-then-parse, retry-with-feedback). That survey, with citations and open questions, lives in its own document:

**→ [docs/methodology-landscape.md](./docs/methodology-landscape.md)**

## 5. Hardware & serving-backend notes

How the technique interacts with consumer/prosumer GPU classes (Pascal P40/P100, Blackwell RTX 5070, Intel Arc B70 / SYCL, Apple M2 Ultra / Metal) and the serving backend (llama.cpp Metal / CUDA / SYCL):

**→ [docs/hardware-notes.md](./docs/hardware-notes.md)**

---

## TL;DR for practitioners

1. **Small instruction-tuned models narrate tool calls under agentic load** — measured ~25% on a real multi-tool write phase. It's structural (weak trained behavior + fragile parser), not a one-line bug.
2. **`tool_choice: required` is not a portable fix.** It's enforced only where the serving parser backs it with a grammar (Qwen3-style native parsers), and ignored where it doesn't (Gemma-4 on the tested build). Verify per model + per build, adversarially.
3. **Grammar-constrained decoding is the durable fix.** It masks illegal tokens before sampling, so narration is impossible. ~25% → 0% over ~45 runs, quality preserved.
4. **Keep grammars loose** (skeleton forced, body free) to dodge the constraint tax.
5. **It's the universal lever** — sampler-level, so it works even where the native tool path is broken (e.g. SYCL builds where the templated path segfaults).

*This is a sanitized field report; empirical numbers are from real runs on the model + hardware classes named above. Your mileage will vary with model family, serving build, and grammar design — measure it.*
