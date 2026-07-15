# The quant axis: aggressive quantization degrades tool-calling independently of parser family

The [main writeup](../README.md) and the [model matrix](./model-matrix.md) establish one axis of
tool-calling reliability: the **maturity of the serving stack's tool parser for that model family**
(Hermes-style native parsers are reliable; the Gemma-4 family parser narrates under load). The matrix's
headline is *"reliability tracks parser family, not model size."*

That is true — but it is **not the whole story.** There is a **second, independent axis: how hard the
weights are quantized.** A model from a *native-reliable* family, served at full or near-full precision,
tool-calls reliably; the **same model, same parser, same harness**, served at an aggressive sub-4-bit
quant, starts to fail. Nothing about the parser family changed — only the number of bits per weight.

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
level tells you whether that reliability survives the quant you actually deployed. You need both — a
native-reliable family served at IQ2_XXS is not a native-reliable deployment.*
