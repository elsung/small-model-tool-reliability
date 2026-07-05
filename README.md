# Making Small Models Reliable at Tool Calling (and Generation, in Production)

**Why small local models narrate tool calls instead of making them, why the fix has to be deployed carefully, and what else breaks once you run this in a real multi-box, multi-backend fleet.**

A field report + methodology survey on getting reliable **tool use**, **structured output**, and
**bounded, safe generation** out of small, locally-served instruction-tuned models (Gemma-4-12B
class, Qwen3-14B class, and abliterated/uncensored creative finetunes) inside an agentic,
multi-tool, multi-turn loop — and, beyond the model behavior itself, out of the harness/serving
plumbing that carries requests to those models in production.

The short version, Part 1 (the core mechanism): on a real multi-tool write phase, a Gemma-4-12B-class model **narrated completion** ("the outline has been written to `…`") instead of emitting the tool call about **25% of the time**. The obvious lever — `tool_choice: required` — does **not** fix it, because the serving stack does not grammar-enforce it for this model family. **Grammar-constrained decoding** does fix it: it masks illegal tokens *before sampling*, so narration is physically impossible. Measured result: **25% → 0% narration over ~45 real runs**, with output quality preserved.

The short version, Part 2 (production hardening — [docs/production-hardening.md](docs/production-hardening.md)): getting the *mechanism* right is not enough. In a real fleet, the grammar-forcing extension from Part 1 turned out to be silently inactive everywhere (a global plugin-discovery gap, not a code bug); the harness driving the model structurally could not forward sampler parameters at all; an uncapped generation on an abliterated creative finetune degenerated into an 88,000-character repetition loop; and a sampler parameter that's completely safe on one backend **crashed and wedged a production multi-node inference cluster** on another. Four more real incidents, four more durable fixes, all folded in below.

---

## Contents

- [1. The problem](#1-the-problem)
- [2. What does *not* work: `tool_choice: required`](#2-what-does-not-work-tool_choice-required)
- [3. What works: grammar-constrained / guided decoding](#3-what-works-grammar-constrained--guided-decoding)
- [4. A tool-calling reliability matrix across models](./docs/model-matrix.md)
- [5. The broader landscape (beyond GBNF)](./docs/methodology-landscape.md)
- [6. Hardware & serving-backend notes](./docs/hardware-notes.md)
- [7. Beyond the grammar: production-hardening the sampling pipeline](./docs/production-hardening.md)
- [8. Instruction adherence: order effects and freedom-scaled judging](./docs/instruction-adherence.md)
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

**Phase-gate *every* phase, including the one that "just reads."** The obvious phases to route onto
the capture/no-tools path are the ones that visibly write an artifact. The phase that's easy to miss
is a **critique / reading / review pass** — a phase whose job is to read accumulated material and
produce feedback or a judgment, not to write a new file. It's tempting to leave that phase on the
native multi-tool path because "it's just reading, there's less that can go wrong." In practice it's
the opposite: a review pass often has the *longest* prompt in the pipeline (it re-reads everything
generated so far — see the [context-management repo](https://github.com/elsung/llm-context-management)
on why per-phase prompt size grows over a job) and the same narration failure mode applies just as
much to "I've reviewed it and it looks good" prose as it does to "the file has been written." **Route
every phase's output through the same forced-structure / no-tools path, and don't carve out an
exception for phases that "only" read** — the read/critique phase is the common holdout that quietly
reintroduces the exact flake this document exists to eliminate.

---

## 4. A tool-calling reliability matrix across models

The result above is one model family. Extending the *same faithful harness* across a fleet of small
local models (Qwen3 9B/14B, Gemma-4-12B, a 3B, creative Llama-3.2 finetunes) shows the pattern
generalizes with a twist: **reliability tracks the model family's serving tool-parser maturity, not its
size** — some models are fine natively and should *not* pay the grammar tax, while others narrate no
matter what and need the grammar (or a reroute). The per-model numbers, verdicts, and how to gate the
grammar on measured need:

**→ [docs/model-matrix.md](./docs/model-matrix.md)**

## 5. The broader landscape (beyond GBNF)

Constrained decoding is one family in a larger toolbox — vLLM/SGLang guided decoding, XGrammar, Outlines, lm-format-enforcer, Microsoft `guidance`, JSON-schema structured outputs — with real tradeoffs (the constraint tax, grammar expressiveness, throughput), and complementary approaches (tool-use fine-tunes, better parsers, ReAct/format-then-parse, retry-with-feedback). That survey, with citations and open questions, lives in its own document:

**→ [docs/methodology-landscape.md](./docs/methodology-landscape.md)**

## 6. Hardware & serving-backend notes

How the technique interacts with consumer/prosumer GPU classes (Pascal P40/P100, Blackwell RTX 5070, Intel Arc B70 / SYCL, Apple M2 Ultra / Metal) and the serving backend (llama.cpp Metal / CUDA / SYCL):

**→ [docs/hardware-notes.md](./docs/hardware-notes.md)**

## 7. Beyond the grammar: production-hardening the sampling pipeline

Getting grammar-constrained decoding (or any before-request injection technique) *working in a lab
test* is a different problem from getting it *reliably active across a real fleet in production*.
Four further incidents, each with a durable fix and the reasoning behind it:

- **An extension can be "shipped" in code and still be inactive everywhere** — global plugin
  discovery is host state, not repo state, and it silently drifted out of sync on every box. Fix:
  ship the extension inside the calling engine's own repo and load it by explicit path, never rely
  on discovery.
- **A harness can structurally lack a field for the sampler parameter you need** — proven by
  capturing a real outgoing request, not by assumption. Fix: an injection hook driven by an
  environment variable, operating below the harness's own (incomplete) request builder.
- **Uncapped generation is a real failure mode** — an abliterated creative finetune degenerated into
  an 88,000-character repetition loop when nothing anywhere actually bounded its generation length,
  even though the "right" repetition-control sampler settings were correctly configured server-side.
  Fix: defense-in-depth — the model class's full recommended sampler profile, an unconditional
  client-side token cap, and an independent server-side generation ceiling.
- **A sampler parameter that's safe on one backend crashed and wedged a production multi-node
  inference cluster on another** (health checks kept reporting healthy while generation hung).
  Fix: a per-provider sampler-capability guard — a drop-and-log filter, empty by default, applied at
  every transport, keyed on the specific deployment rather than a generic backend label.

**→ [docs/production-hardening.md](./docs/production-hardening.md)**

## 8. Instruction adherence: order effects and freedom-scaled judging

A downstream problem from everything above: even once a small model reliably produces well-formed
output, does that output actually **honor a constrained-transform instruction**, or does it quietly
default back to preserving what it was told to change? Two more findings, both measured: (1)
**instruction order, not length or emphasis, controls whether a small model executes an
identity/rename transform** — the same requirement placed first and fused to the primary verb got
0/9 leaks of the source's own names; placed last, after a stack of other negations, got 76–184
leaks per run, even though every *world/lore* negation held regardless of position; (2) an automated
adherence judge that pauses on any drift is worse than none — the fix is a **freedom-scaled** judge
that classifies instruction elements as HARD (must hold regardless of creative freedom granted) vs.
SOFT (invited to transform), and only ever pauses on a HARD violation or total-intent abandonment,
scaled to how much freedom the job actually granted.

**→ [docs/instruction-adherence.md](./docs/instruction-adherence.md)**

---

## TL;DR for practitioners

1. **Small instruction-tuned models narrate tool calls under agentic load** — measured ~25% on a real multi-tool write phase. It's structural (weak trained behavior + fragile parser), not a one-line bug.
2. **`tool_choice: required` is not a portable fix.** It's enforced only where the serving parser backs it with a grammar (Qwen3-style native parsers), and ignored where it doesn't (Gemma-4 on the tested build). Verify per model + per build, adversarially.
3. **Grammar-constrained decoding is the durable fix.** It masks illegal tokens before sampling, so narration is impossible. ~25% → 0% over ~45 runs, quality preserved.
4. **Keep grammars loose** (skeleton forced, body free) to dodge the constraint tax.
5. **It's the universal lever** — sampler-level, so it works even where the native tool path is broken (e.g. SYCL builds where the templated path segfaults).
6. **The mechanism being correct doesn't mean it's active** — verify any extension/hook is actually loaded on every host, not just that the code to load it exists; prefer ship-with-engine + explicit-path-load over global plugin discovery.
7. **Harnesses can silently lack a sampler surface** — capture and inspect a real outgoing request before trusting that a config value has any effect; inject below the harness via an env-var-driven hook when it doesn't.
8. **Uncapped generation is a production incident, not a theoretical risk**, especially on abliterated/uncensored finetunes — cap client-side *and* server-side, independently, and apply the model's full recommended sampler profile.
9. **Sampler-parameter safety is per (backend, version, decoding-mode)** — the same "extra" parameter can crash, silently no-op, or work cleanly on different deployments of the same backend software. Guard with a default-empty, drop-and-log capability filter at every transport.
10. **Parse small-model output defensively — they're verbose and decorative.** Small models routinely
    wrap their real output in extra, unbalanced markdown fences (banner-style ASCII art, decorative
    headers). Naive "truncate at the first closing fence" extraction logic can wreck a perfectly good
    document. See [docs/production-hardening.md §5](docs/production-hardening.md#5-small-models-are-decorative--pair-your-fences-dont-just-count-them).
11. **Instruction order controls whether a small model executes a constrained transform, decisively
    more than the instruction's length or emphasis does** — lead with the identity/rename transform,
    fused to the primary verb, as one absolute-scope clause; don't bury it under a stack of other
    negations. Measured 0/9 leaks with that ordering vs. 76–184 leaks/run with the identical
    requirement placed last. See [docs/instruction-adherence.md §1](docs/instruction-adherence.md#1-instruction-order-controls-adherence-decisively).
12. **An adherence judge must be freedom-scaled, not zero-tolerance.** Classify instruction elements
    HARD (must hold) vs. SOFT (invited to transform); pause only on a HARD violation or total-intent
    abandonment, scaled to how much creative freedom the job granted. Bold transformation of a SOFT
    element is success, never drift. See [docs/instruction-adherence.md §2](docs/instruction-adherence.md#2-freedom-scaled-adherence-judging).

## Related work

This repo covers **reliability of the generation mechanism itself** (tool calls, structured output,
bounded/safe sampling) — and, since §8, whether the *content* that mechanism produces actually
adheres to a constrained instruction once it's producing well-formed output. Its sibling repo,
**[llm-context-management](https://github.com/elsung/llm-context-management)**,
covers the companion problem — **keeping the *input* to that mechanism inside a real, working context
budget** as pipelines accumulate state across phases (the same phased pipelines this repo's grammar
and sampler fixes are applied to). The two compose: a phase can fail because the model narrated
instead of writing (this repo), or because its prompt silently overflowed the box it was routed to
(the context repo) — both failure modes look identical from the outside ("the job produced garbage or
nothing"), so debugging either one should check both. §8's freedom-scaled adherence judge is a
direct extension of that sibling repo's `freedom_lever`/directive concepts
([docs/04 §4.2](https://github.com/elsung/llm-context-management/blob/main/docs/04-goal-aware-extraction.md#42-the-goal-triple--what-parameterizes-relevance))
into a judging policy that should share the same job-level setting, not redefine "how much freedom"
independently.

*This is a sanitized field report; empirical numbers are from real runs and real incidents on the model + hardware classes named above. Your mileage will vary with model family, serving build, backend version, and deployment topology — measure it, don't assume it.*
