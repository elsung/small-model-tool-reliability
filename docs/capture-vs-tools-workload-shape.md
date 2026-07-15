# Capture vs. tool-calling: route by workload shape, not by model name

A tempting thesis: *small models are unreliable at tool-calling, so stop letting them call tools —
make the model a pure text generator, capture its completion, and let deterministic code own every
file write.* ("Capture mode": run the agent with `--no-tools`, the completion **is** the artifact,
your code writes it to disk.)

**The thesis is right for a narrower reason than its usual justification — and its usual
justification is false.** Tool-calling is not inherently unreliable. What collapses is the *loop*,
not the *call*. Route on that distinction and you get the win without the overreach.

> Labels: `[MEASURED-RAW]` (primary data pulled directly) · `[PEER-REVIEWED]` · `[VENDOR]`.

## 1. "Tool-calling is inherently unreliable" is refuted

The Berkeley Function-Calling Leaderboard publishes the same weights under **two harnesses** —
native function-calling (FC) vs prompt-only — which is exactly the controlled comparison this
argument needs. `[MEASURED-RAW]` — pulled from
[gorilla.cs.berkeley.edu/data_overall.csv](https://gorilla.cs.berkeley.edu/data_overall.csv):

| Model | Overall | **Single-turn AST** | Multi-Turn |
|---|---|---|---|
| Claude-Opus-4-5 **(FC)** | **77.47%** | 88.58% | **68.38%** |
| Claude-Opus-4-5 **(Prompt)** | **33.47%** | **89.65%** ← | **16.12%** |
| Claude-Haiku-4-5 **(FC)** | 68.70% | 86.50% | 53.62% |
| Claude-Haiku-4-5 **(Prompt)** | 25.26% | 55.42% | 1.75% |

Overall, tools win enormously — **77.47 vs 33.47**. Taking tools away from a capable model is
catastrophic. Anyone claiming "just remove the tools" as a general architecture is contradicted by
their own leaderboard.

## 2. …but the single-turn column inverts it

Look again at **Single-turn AST: prompt mode wins, 89.65 vs 88.58.**

The entire 77→33 collapse lives in the *agentic* categories — **Multi-Turn 68.38 → 16.12**,
**Memory ~74 → ~2**, **Web Search ~85 → ~13**. For Haiku, multi-turn prompt mode scores **1.75%**:
a total floor.

> **It is the loop that fails without tools, not the call.** A single "read this brief, produce
> this document" turn does not need a tool loop — and measurably does slightly *better* without one.

That is the real, defensible case for capture mode, and it is a statement about **workload shape**,
not about model quality.

## 3. The second condition: open weights

The **[format tax](./format-tax-and-generate-then-conform.md)** — the accuracy lost to
format-requesting instructions — is documented as largely an **open-weight phenomenon**
(*"most recent closed-weight models show little to no format tax"*,
[arXiv:2604.03616](https://arxiv.org/abs/2604.03616) `[PEER-REVIEWED]`).

So capture-first is indicated when **both** conditions hold:

| Condition | Why |
|---|---|
| **Single-shot artifact generation** (not a multi-turn loop) | BFCL §2 — prompt mode ties or wins here, and only here |
| **Open-weight / locally-served model** | the format tax and the narration flake concentrate here |

Neither is a law of nature. Both are properties of a *deployment*, and both are checkable.

## 4. The failure this fixes has a peer-reviewed name and number

**CODESTRUCT** — Amazon, ACL 2026 main. `[PEER-REVIEWED]`
[arXiv:2604.05407](https://arxiv.org/abs/2604.05407). SWE-bench Verified:

| Model | Baseline → Structured | Empty patches |
|---|---|---|
| **GPT-5-nano** | **19.6% → 40.4%** (+20.8 pp) | **46.6% → 7.2%** |
| GPT-5 | 66.0% → 67.2% (+1.2 pp) | — |
| **Qwen3-8B** | **13.2% → 13.0% (−0.2 pp)** | recovered 41, **gained zero** |

An **empty patch** is *"a failure to externalize an intended edit rather than incorrect high-level
reasoning."* That is the rigorous name for **narrate-don't-write**: the model *did* the thinking and
then failed to emit it through the tool. At **46.6%** for a small model, it is the dominant failure
mode — and restructuring the output channel takes it to **7.2%**.

**The Qwen3-8B row is the warning.** Fixing the wrapper recovers work *only if the content
underneath was good*. It recovered 41 patches and gained **zero** accuracy.

> **Calibrate expectations: capture mode and output-aware success recover work you were already
> throwing away. They do not make the model smarter.** If the underlying generation is weak, a
> better channel surfaces weak output faster. That is still worth having — silent loss is worse
> than visible mediocrity — but do not sell it as a quality win.

## 5. Where the thesis breaks — when NOT to do this

- **Multi-turn / agentic loops.** BFCL is unambiguous: prompt-only multi-turn is a floor (Haiku
  **1.75%**). If the task genuinely needs read→decide→act→observe, use tools, on a model that can.
- **Exploration, memory, web search.** ~74→~2 and ~85→~13. Tool-choice *is* the reasoning here.
- **Capable closed-weight models.** They pay little format tax and are strong tool-callers. Capture
  mode buys you nothing and costs you the loop.
- **Outputs exceeding max_tokens.** A single completion can't hold a book. Chunk it (per-episode
  capture with a token cap scaled to the chunk count) or stay agentic.
- **Don't over-rotate into anti-agent ideology.** The flagship anti-agent result's own authors
  defected: Xia & Zhang's [Live-SWE-agent](https://arxiv.org/abs/2511.13646) scores **77.4%** vs
  Agentless's **50.8%**. The people who best made the "agents are unnecessary" case now ship an agent.

## 6. The design that follows

**Split "tool use" into three things and decide each separately** — the common error is treating
them as one:

| | Give it to the LLM? | Why |
|---|---|---|
| **Effects** (write/edit the artifact) | **No** — model emits text, code writes it | zero model advantage; this is where narration/non-termination bite |
| **Input assembly** (fetch the brief, the reference material) | **No** — just put it in the prompt | deterministic, cheaper, and removes a whole failure class |
| **Control flow** (what to do next, when to stop) | **Depends on shape** — single-shot: no. Genuine exploration: **yes, keep tools** | this is the column BFCL says you cannot take away |

The thesis conflates all three. Flip the first two; **leave the third alone.**

## 7. Handling the residue: noise, and broken markers

Capture mode's honest objection: the completion may carry preamble ("Sure! Here you go:"), thinking,
or a missing closing fence.

- **Preamble is eliminated by construction, not discouraged.** A grammar anchoring the first token
  to `#` makes "Sure" *unsampleable* — probability zeroed before sampling. See the
  [main writeup](../README.md).
- **Thinking is a separate, delimited channel** — strip it, or disable reasoning for the turn.
- **Broken markers are a parsing problem, not a loss.** Use a deterministic ladder: strict fence →
  tolerant (opener present, closer missing — by far the most common) → structural (first heading →
  EOF) → **validate the extraction** (expected section count, length floor) → retry naming the
  specific defect → loud fail. **Never silently accept.**

The asymmetry is the whole point:

> **A tool-call failure is silent and lossy — the work vanishes, `rc=0`, no file. A capture failure
> is loud and recoverable — the text is right there in a string you can strip, repair, validate,
> and unit-test against a corpus of real outputs.** Worst-case capture beats best-case tool-failure.
> And you cannot unit-test "did the model decide to call the tool this time."

## 8. Should the deterministic pipeline be in Rust?

**No.** GPU inference dominates by ~3 orders of magnitude — roughly **40 µs** of file/parse work
against **~18,500 µs per token** of generation. The win from this architecture is **determinism, not
speed**, and Python delivers 100% of the determinism. Rust buys no measurable wall-clock and costs a
second toolchain.

## 9. Takeaways

1. **Don't justify capture mode with "tools are unreliable."** BFCL refutes it (77.47 vs 33.47).
   Justify it with **workload shape** (single-shot) **× weight tier** (open).
2. **Route on those two axes.** A model-name regex (`gemma|llama-3|…`) can see *neither* — it will
   mis-route a strong model on an agentic task and a weak quant on a single-shot one.
3. **Take effects and input-assembly away from the model. Leave control flow alone** unless the
   workload is genuinely single-shot.
4. **Expect recovered work, not better work** (CODESTRUCT, Qwen3-8B: +41 patches, +0.0 accuracy).
5. **Check [Tool Suppression](./format-tax-and-generate-then-conform.md#7--tool-suppression--grammar-and-tools-can-be-mutually-exclusive)
   before combining a grammar with an active tool loop** — the masks can make your terminal token
   unreachable. Capture mode sidesteps this entirely by having no tools to suppress.
