# Hardware & serving-backend notes

Constrained decoding is a *software* technique, but where it runs matters — the serving backend (llama.cpp Metal / CUDA / SYCL, or vLLM/SGLang on CUDA) determines which paths are even available. This doc collects practitioner nuance across the consumer/prosumer GPU classes we exercised. None of it is exotic hardware; it's the kind of mixed fleet a small team or a home lab actually runs.

## The key invariant

> **Native tool calling** depends on the serving stack's template-rendering + tool-parsing pipeline. **Grammar-constrained decoding** depends only on the sampler. The sampler is present in every mainstream build; the tool pipeline is not always healthy on every backend.

That single fact is why grammar is the portable lever: it degrades gracefully to "still works" on backends where the templated-tool path is broken.

## Backend-by-backend

### Apple Silicon — M2 Ultra, llama.cpp Metal
- The reference "good regime" for this write-up. A recent build with template rendering enabled (`--jinja` on) and the model served with **thinking/reasoning disabled** for tool-critical turns.
- Native tool calling *works* here (the fragile parser is present but functional), yet still showed ~25% narration flake on the real multi-tool loop — this is the model-behavior layer failing, not the backend.
- Grammar-constrained decoding: **100% valid** (20/20, plus 45/45 across three phases). No Metal-specific issues; grammar sampling is CPU-side mask logic and is Metal-safe.
- Unified memory makes large-context (64k+) serving of a 12B model comfortable, which is what let us reproduce the failure at realistic prompt sizes.

### NVIDIA Blackwell — RTX 5070, llama.cpp CUDA
- Modern consumer card; CUDA build with template rendering on by default. Native tool path healthy; grammar path healthy.
- Nothing special required — this is the "everything works, just keep the grammar loose" case. Also the class where vLLM/SGLang + XGrammar guided decoding is a natural fit if you outgrow llama.cpp.

### NVIDIA Pascal — P40 / P100, llama.cpp CUDA
- **Memory-bound and FP16-caveated.** Pascal predates good half-precision throughput (no usable FP16 tensor path; P40 FP16 is famously crippled), so these cards are effectively **dense-only, INT8/FP32-leaning, and slow** for anything heavy.
- Practical consequence for *this* topic: they can serve small dense models and run the grammar sampler fine, but they are **too slow for heavy generation** phases. Grammar sampling adds negligible overhead, so it does not change the calculus — but the card class limits you to smaller models regardless.
- Guidance: if a Pascal box is in the fleet, give it the *smallest* viable model for a phase and lean on a loose grammar for reliability; don't expect it to carry large-context writer generation. MoE / speculative tricks give little help on Pascal.

### Intel Arc — B70, llama.cpp SYCL
- **The most instructive case.** On the SYCL build, the templated-tool path (`--jinja`) **segfaults / is effectively unavailable.** That means: native tool calling is off, and `tool_choice` has nothing to enforce it → the model **narrates by default** (guaranteed, not just 25%).
- **Grammar-constrained decoding still works**, because it is sampler-level and never touches the jinja/tool-parsing pipeline. This is the headline: the SYCL box goes from "capture-only / can't do reliable tool use" to "can produce forced-structured output natively," purely by using a per-request grammar.
- If you have a SYCL box in the loop, **route tool-heavy phases either (a) off it to a CUDA/Metal box, or (b) through the grammar path.** Don't rely on native tool calling there.

## Fleet summary

| backend | native tool calling | `tool_choice: required` (Gemma-4) | grammar / `response_format` |
|---|---|---|---|
| Metal (M2 Ultra) | works, ~25% narration flake | ignored | **works — 100%** |
| CUDA (RTX 5070) | works | ignored | **works** |
| CUDA (Pascal P40/P100) | works but slow; small models only | ignored | **works** (grammar overhead negligible) |
| SYCL (Arc B70) | templated path segfaults → narration by default | n/a | **works — sampler-level, jinja-independent** |

## Serving-hygiene checklist (independent of the grammar fix)

Even if you adopt grammar-constrained decoding, get the native path healthy where you can — it's your fallback and it matters for genuine multi-tool loops:

1. **Enable template rendering** (`--jinja` on llama.cpp) *explicitly*. The default is build-dependent; pin it so a future package update can't silently disable your tool path.
2. **Disable thinking/reasoning for tool-critical turns.** A tool call emitted inside a thinking block may be routed into a separate `reasoning_content` channel and never surface as a `tool_call`. Several stacks also silently bypass grammar/`response_format` enforcement when thinking is enabled.
3. **Verify your build post-dates your model family's tool-parser fixes.** Small-model tool parsers are actively churning; an older build can corrupt args or mis-detect the format.
4. **Avoid aggressive KV-cache quantization** on tool-critical serving — it degrades tool-call formatting.
5. **Prefer a model with a mature, strict tool parser** (Qwen3-style Hermes native format) when the phase is a genuine multi-tool loop and model choice is free. Reserve the grammar path for single-output "produce this artifact" phases.
6. **Watch for strict role-alternation chat templates on community finetunes/merges.** Some community-authored GGUF chat templates (observed on DavidAU-style abliterated/creative merges) hard-enforce strict user/assistant turn alternation in the template itself — independent of whether `--jinja` template rendering is on or off. A multi-turn tool round-trip (assistant → tool result → assistant again) doesn't fit that strict alternation shape, and the serving stack's tool-parser auto-detection can misfire on it in ways that look like a jinja/template-flag bug but aren't — the template is doing exactly what its author wrote, it just wasn't written with multi-turn tool loops in mind. This is a second, independent reason (beyond narration/parser-fragility, see the [main writeup](../README.md)) to route these checkpoints through the no-tools/capture or grammar path rather than the native multi-turn tool loop: verifying `--jinja` and the tool parser doesn't rule out a template-shape mismatch underneath both.
