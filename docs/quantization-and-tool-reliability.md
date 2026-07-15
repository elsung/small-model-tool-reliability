# The quant axis: aggressive quantization degrades tool-calling independently of parser family

The [main writeup](../README.md) and the [model matrix](./model-matrix.md) establish one axis of
tool-calling reliability: the **maturity of the serving stack's tool parser for that model family**
(Hermes-style native parsers are reliable; the Gemma-4 family parser narrates under load). The matrix's
headline is *"reliability tracks parser family, not model size."*

That is true — but it may not be the whole story. There appears to be a **second axis: how hard the
weights are quantized.** A model from a *native-reliable* family, served at full or near-full precision,
tool-calls reliably; the **same model, same parser, same harness**, served at an aggressive sub-4-bit
quant, starts to fail. Nothing about the parser family changed — only the number of bits per weight.

> ## ⚠️ Evidence status — read before citing this page
>
> **This is a field observation, not an established result.** Be clear-eyed about what backs it:
>
> - **What we observed:** one deployment (below). n=1, uncontrolled, no A/B at matched prompt scale.
> - **What the literature actually says:** we could find **no published measurement of IQ2_XXS — or
>   any i-quant — against a function-calling benchmark.** That is a genuine gap, not a search failure.
>   The nearest rigorous datapoint is [LlamaRestTest](https://arxiv.org/html/2501.08598v1)
>   `[PEER-REVIEWED]`: Llama3-8B generating valid structured API parameters scores **72.44% at full
>   precision → 66.12% (Q8) → 60.21% (Q4) → 29.12% (Q2)**, with the loss attributed to *invalid*
>   responses — the right failure mode, but **legacy llama.cpp 2-bit, not IQ2_XXS**, and a structured-
>   output proxy rather than tool-calling. Treat it as an upper bound on the damage, not an estimate.
> - **Corroborating but indirect:** Unsloth's published GGUF metrics show **KLD99.9 ≈ 4.22 at IQ2_XXS
>   vs ≈ 0.41 at Q4_K_XL** (~10×) — tail-token divergence is exactly what would break a terminal
>   tool-call token, but it is a proxy, not a measurement of tool-calling.
> - **FP8 is measured and safe.** NVIDIA's Nemotron-3-Nano BF16→FP8 shows ≤3.2 pt drops on tau-bench
>   and −0.61 on BFCL v4; [KAMI](https://arxiv.org/html/2511.08042) finds FP8 *beating* full precision
>   on Qwen3-14B/32B agentic tasks. **4-bit weight-only against tool-calling is simply unmeasured.**
> - **llama.cpp upstream does not corroborate it.** Its only statement is an unsourced line in the
>   function-calling docs warning about *KV-cache* quant (`-ctk q4_0`), with no data. No upstream issue
>   links weight bit-width to tool-calling failure; upstream's real failures are parser/grammar
>   contract bugs and tool fan-out.
> - **Do not cite [CarbonCall](https://arxiv.org/html/2504.20348) for this.** It is widely read as
>   evidence that quantization degrades function calling. It contains **no such measurement** — the
>   claim appears only as unsupported motivating text.
>
> **Honest summary: "aggressive quant degrades tool-calling" is well-motivated and consistent with the
> adjacent evidence, but unverified for i-quants specifically.** The engineering advice below is
> defensive and cheap, so it stands on its own regardless. The claim does not.

## The observation

A large hosted-tier Mixture-of-Experts model — the kind that is a **strong, reliable native tool-caller
at FP8/BF16** on the big hosted tier — was served locally at an aggressive **~2-bit importance-matrix
quant (IQ2_XXS)** to fit a constrained VRAM budget. On the same agentic multi-tool loop where it is
flawless at FP8, the IQ2_XXS build:

- flailed in native tool-calling (mangled or mistimed calls), and
- frequently **never emitted end-of-turn** — it kept generating past the point where it should have
  stopped, running the agent turn into the harness timeout and getting killed (`rc=124`).

Only the quantization changed. The base model, the parser family, the template, and the tool set were
identical to a configuration that works. This is the same *class* of failure the matrix documents for
fragile-parser families — narration / non-termination instead of a clean call-then-stop — reached by a
**different road**: not a weak parser, but weights too lossy to reliably execute the precise token
sequence a tool call requires.

## Why quant erodes tool-use faster than perplexity suggests

Perplexity and most benchmark scores are **averages over fluent next-token prediction**, and fluent
prose is robust to quantization — you can strip a model to 2–3 bits and it still reads well. **Tool
calling is not average-case fluent prediction.** It depends on the model reliably hitting a *specific,
low-probability-under-prose* token pattern at exactly the right moment: emit the tool-call opening
tokens *now* (not more prose), fill structured arguments, then emit the terminal / end-of-turn token
*and stop*. That is precisely the kind of sharp, discriminative behavior that low-bit quantization
blurs first.

So the degradation is **hidden**: a model's perplexity and its chat quality can look completely fine at
IQ2_XXS while its tool-calling and structured-output reliability have already collapsed. Community and
llama.cpp guidance converge on the same warning — *tool use and structured output degrade faster than
perplexity implies*; **Q4_K_M-class weights are the safe floor, and sub-Q4 (IQ2_XXS / Q2_K) plus
KV-cache quantization is the danger zone** for anything that must call tools or emit strict structure.
KV-cache quant (e.g. a 4-bit KV) compounds it — keep the KV cache at f16/q8 when tool reliability matters.

## The corrected model: two axes, not one

Reliability is the product of **both** axes:

| | native-reliable parser family (Hermes-style) | fragile parser family (Gemma-4 style) |
|---|---|---|
| **Q4_K_M or better** | reliable native tool-calling | narrates under load → needs grammar |
| **aggressive quant (IQ2_XXS / Q2_K)** | **degrades toward unreliable** — the quant axis bites even here | reliably bad — needs grammar |

The matrix's original claim ("not size — it's the parser family") stands *within a fixed quant*. Add the
quant axis and the full rule is: **a model needs the grammar path if EITHER its parser family is fragile
OR it is aggressively quantized** — regardless of how reliable that same architecture is at full
precision on the hosted tier. Size is still not the predictor; parser-family **and** bit-width are.

## Practical consequences

1. **Gate the grammar on `(fragile parser family) OR (aggressive quant)`, not on size and not on the
   model's full-precision reputation.** A big MoE that is a gold-standard native tool-caller at FP8 must
   still be routed to the grammar-forced path when you serve it at IQ2_XXS. Detecting this is cheap: the
   quant is almost always in the model id or the provider tag (`iq1`, `iq2`, `q2_k`, `xxs`) — match on it
   and add that build to the grammar set, even though the same base model at Q4+ stays on the native path.
2. **Keep aggressive quant for *generation*, not for reliable tool / structured calls** — unless those
   calls are grammar-constrained. IQ2_XXS is a fine way to fit a big creative-generation model in tight
   VRAM; it is a poor choice for the model's tool-dispatch or structured-output phases unless a grammar
   backstops them.
3. **A terminal grammar fixes the non-termination mode specifically.** The failure here is not only
   "narrate instead of call" — it is also "call (or ramble) and never stop." A grammar whose `root`
   matches **exactly one** structured object and then permits **only** the end-of-sequence token makes
   running past the object *physically impossible*: the turn terminates deterministically instead of
   drifting into the timeout. (Lazy variants — constrain only after a trigger token — keep reasoning
   free while still forcing termination. See the [landscape doc](./methodology-landscape.md).)
4. **Watch the KV cache.** If you quantize the KV cache alongside sub-Q4 weights, you are stacking two
   degraders on the exact behavior you need. Keep KV at f16/q8 on any path that must call tools reliably.

## The one-line version

*Parser-family maturity tells you whether a model tool-calls reliably at a given precision. Quantization
level may tell you whether that reliability survives the quant you actually deployed — a native-reliable
family served at IQ2_XXS should not be assumed to be a native-reliable deployment. Verify it on your own
build; nobody has published the measurement.*

## The measurement nobody has run

If you want to contribute something genuinely new here: **sweep one model across FP16 / Q8 / Q4_K_M /
IQ2_XXS against a real function-calling benchmark (BFCL, tau-bench) at realistic prompt scale, holding
the parser, template, and tool set fixed.** As of mid-2026 that measurement does not exist publicly.
Everyone — including this page — is extrapolating from perplexity, KL divergence, or adjacent
structured-output proxies. Run it at your real artifact size, not on a toy prompt: the
[matrix](./model-matrix.md) documents a model that is clean at reduced scale and flakes ~25% at full
scale, so a small-prompt sweep will under-report.
