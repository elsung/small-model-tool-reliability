# Beyond the grammar itself: production-hardening the sampling pipeline

The [main writeup](../README.md) establishes that grammar-constrained decoding is the durable fix
for small-model tool/structured-output narration. Running that fix — and small local models more
generally — in an actual multi-box, multi-backend production fleet surfaced four further failure
modes that have nothing to do with the grammar mechanism itself and everything to do with the
plumbing around it: whether the fix is *actually active* in production, whether the harness can
even carry sampler settings at all, what happens when generation runs uncapped, and what happens
when a sampler parameter that's perfectly normal on one backend is unsafe on another. Each was a
real incident, not a hypothetical. This document folds those into the reliability picture as a
second layer: **getting the mechanism right (Part 1) is necessary but not sufficient — you also
have to get its *deployment* right.**

---

## Contents

- [1. Ship the extension with the engine — don't rely on global plugin discovery](#1-ship-the-extension-with-the-engine--dont-rely-on-global-plugin-discovery)
- [2. Harnesses can't always send sampler params — you need an injection mechanism](#2-harnesses-cant-always-send-sampler-params--you-need-an-injection-mechanism)
- [3. Uncapped generation is a real failure mode, not a theoretical one](#3-uncapped-generation-is-a-real-failure-mode-not-a-theoretical-one)
- [4. Per-backend sampler-capability guard — a crash, not just a warning](#4-per-backend-sampler-capability-guard--a-crash-not-just-a-warning)

---

## 1. Ship the extension with the engine — don't rely on global plugin discovery

### The failure

Several agent CLIs support a hook/extension mechanism that lets calling code intercept and modify
the outgoing request before it hits the model server — exactly the seam you need for
grammar-injection (Part 1) or sampler-injection (§2 below). The natural way to install such an
extension is through the CLI's own plugin-discovery mechanism: put the extension file somewhere the
CLI scans on startup, or register it through a one-time "install a package" command, and the CLI
picks it up automatically on every invocation from then on.

**In a real multi-box fleet, this silently didn't work anywhere.** A `list installed
extensions`-equivalent command, run on every box that was supposed to have the grammar-forcing
extension active, reported either nothing installed or only an unrelated extension — on every box.
The extension file existed in source control and had been "shipped" in the sense that the code that
was supposed to load it existed and was merged, but the actual runtime artifact was never installed
on any serving host. Every box had been silently running the *fallback* path (best-effort
freeform-text capture, not the grammar-forced path) the entire time, and nothing in normal operation
signaled this — the fallback path is designed to degrade gracefully, so the pipeline kept producing
output, just without the reliability guarantee it was supposed to have. This class of failure is
insidious precisely *because* the fallback is well-designed: a fallback that fails loudly would have
been caught immediately; a fallback that "just works, a bit worse" can go unnoticed indefinitely.

### Why this happens

Global plugin-discovery directories are **host state**, not **repo state**. Whether an extension is
actually present on a given box depends on whether someone ran a one-time install step on that
specific box, at some point, and whether that state has drifted since (a host rebuild, a fresh box
added to the fleet, a home-directory reset) — none of which is tracked anywhere your deployment
process actually looks. A code review of the calling logic looks completely correct ("if the
extension is present, do X; otherwise fall back to Y") — the code's own comments even documented the
fallback as intentional and safe. The comment was *true* in isolation and *dangerously incomplete*
in practice, because it made "no extension installed" sound like an acceptable edge case rather than
the actual, universal, silent status quo.

### The fix — ship-with-engine, explicit-load

Stop depending on the CLI's own plugin registry entirely. Instead:

1. **Ship the extension file as a normal file inside the calling engine's own repository** (a
   `pi-extensions/`-style directory, or wherever your project keeps its own assets) — not in any
   directory the harness CLI scans on its own. A `git pull` / redeploy of the calling engine
   deploys the extension automatically, with zero separate install step and zero dependency on
   host-local package-manager state.
2. **Load it with an explicit path flag on every invocation** (most agent CLIs that support
   extensions/hooks also support loading one by an explicit file path, independent of the
   discovery mechanism — e.g. `<cli> -e /path/to/extension.ts ...`). This makes activation a
   property of *the call you make*, not a property of *what happens to be installed on this host
   right now*.
3. **Verify the extension actually loaded**, don't just verify the flag was passed. A cheap
   smoke check (load the extension with a trivial no-op invocation and confirm a clean exit code,
   or have the extension emit a detectable marker on load) catches a broken or incompatible
   extension file at deploy time instead of degrading silently to the fallback path in production.

**Why this is the durable pattern, generalized:** any mechanism whose activation depends on
per-host installed state rather than per-deploy shipped state will drift out of sync with your code
across a fleet of more than one machine, and it will do so silently if (as is common, and good
practice on its own) the calling code has a graceful fallback for "not installed." **Make activation
deploy-coupled, not host-state-coupled**, and treat any extension/plugin/hook system in your stack
that relies on host-local discovery with the same suspicion.

---

## 2. Harnesses can't always send sampler params — you need an injection mechanism

### The problem

A harness that wraps a model API doesn't necessarily forward the full sampler surface a serving
backend supports. One widely-used agent CLI, when driving a locally-served OpenAI-compatible
endpoint, was confirmed (by reading its own request-building source, not by guessing) to forward
**only `temperature`, and only when explicitly configured** — no field exists anywhere in its
request schema or request-building code for `repeat_penalty`, `top_k`, `min_p`, or any of the
DRY-family repetition-control parameters. This was proven directly, not inferred: a captured live
outgoing HTTP request body from a real invocation contained only protocol-level fields (model,
messages, streaming flags, tool definitions) — **no sampler keys were present at all**, confirming
the harness structurally cannot send them, not merely that it wasn't configured to.

This matters a great deal for small local models specifically (see §3) — some model families need
active repetition control to stay usable, and if your harness has no schema field for it, setting
the "right" value somewhere in your config does nothing, because there is no code path that ever
puts it on the wire.

### The fix — inject below the harness, from an environment variable

The same before-request hook mechanism used for grammar injection (Part 1, and §1's ship-with-engine
pattern) is the right seam here too: a hook that runs immediately before the harness sends its
request, reads a JSON blob of desired sampler settings from an environment variable set by the
calling process, and **merges it directly into the outgoing request body** — bypassing the harness's
own (incomplete) request-building code entirely.

```
calling process:
  env[SAMPLER_SETTINGS_JSON] = json({"repeat_penalty": 1.1, "min_p": 0.05, ...})
  invoke harness CLI, explicitly loading the injection hook (per §1)
        │
        ▼
harness's own request builder assembles {model, messages, ...}  (no sampler fields — it can't)
        │
        ▼
injection hook (before-request) reads env[SAMPLER_SETTINGS_JSON], merges into the request body
        │
        ▼
sent to the model server — now WITH the sampler fields the harness itself never learned to send
```

This decouples two concerns that are easy to conflate: **"what sampler settings does this specific
job need"** (an orchestration/config-layer decision, made per job or per model) from **"can the
harness's own API surface express it"** (irrelevant once you're injecting below that layer). The
calling layer owns the settings; the hook is a dumb, generic merge — it doesn't need to know
*why* a given value was chosen, just that it should land in the request.

**Practical corollary:** don't assume any given harness/CLI forwards a sampler field just because the
underlying API it wraps supports it. Verify per harness, per version, the way the "does `tool_choice`
work" question in Part 1 had to be verified per model/parser — capture the actual outgoing request
body for a real invocation and inspect it directly. A config value that silently has no effect,
because nothing in the call chain ever transmits it, is a much worse failure than an explicit error
would be, because it looks like it should be working.

---

## 3. Uncapped generation is a real failure mode, not a theoretical one

### The incident

A creative/roleplay-finetuned small model (a DavidAU-style abliterated Llama-3.2 merge, served
locally) degenerated on a real outline-generation job into a single repeated token/phrase for
roughly **88,000 characters** before the job crashed. Separately, a "keep this tight and
glance-able, it's reference material, not prose" phase on the same model class produced an output
file **~10x the size a well-behaved run produces** — which then cascaded into a second-order
failure downstream, because that oversized artifact got inlined into the *next* pipeline step's
prompt and blew that step's own context budget.

### Root-cause discipline: disprove the tempting wrong hypothesis first

The obvious first hypothesis — "the harness is overriding the server's configured repetition-control
settings, silently resetting them to a no-op" — turned out to be **wrong**, and was disproven with
direct evidence rather than accepted on plausibility:

1. The server's own launch configuration, on every box serving this model, already had the model
   family's own vendor-recommended sampler settings correctly baked in (temperature, repeat
   penalty, a bounded repeat-lookback window, top-k/top-p/min-p, and a full DRY-family repetition
   penalty configuration) — verified byte-for-byte against the model's own published documentation.
2. A captured live outgoing request (§2's technique) confirmed the harness sends **no sampler
   fields at all** — so it is structurally impossible for it to have "overridden" anything; there
   was nothing to override.
3. The server's own reported default-generation-settings (queried live) exactly matched its launch
   configuration — confirming the server was correctly falling back to its own defaults when the
   request omitted sampler fields, exactly as it should.

**The real root cause was the §1 failure, compounding with a second independent gap**: the
extension that was supposed to enforce a bounded generation length for these capture-mode phases was
never actually active (same silent-fallback pattern as §1), *and*, independently, the model server's
own hard generation-length ceiling flag had been left at its default of "unbounded." Two things that
each looked individually acceptable ("the client-side cap should handle it," "the server allows
long generations for phases that need them") combined into "nothing anywhere actually bounded
generation length" for this model class.

### Why this model class specifically needs both correct sampler settings AND a hard length cap

Abliterated/uncensored creative finetunes are qualitatively more prone to falling into repetition
loops or open-ended rambling than mainstream instruction-tuned models, even under correctly
configured repetition-control settings — the finetuning process that removes refusal behavior can
also thin out the more general training signal that keeps mainstream models self-terminating and
on-topic. Vendors that publish these merges typically publish **class-specific recommended sampler
profiles** (e.g., a "Class 3" profile: moderate temperature, a real repeat-penalty with a bounded
lookback window, top-k/top-p/min-p all set, plus a DRY-family configuration and small presence/
frequency penalties) — and for this model class, using them is not optional polish, it's the
difference between usable output and the repetition-loop failure mode above.

But **correct sampler settings are not sufficient on their own if nothing caps generation length.**
A model that's merely *prone to occasionally* degenerate, running with no ceiling on how many tokens
it's allowed to produce, will eventually run the failure to its full, uncapped extent instead of
being cut off early. The two are complementary, not substitutes for each other.

### The fix — defense in depth, verified live

1. **A client-side `max_tokens` cap**, sized to what the phase actually needs (a "keep it tight,
   it's reference material" phase should cap far lower than a long-form prose phase), applied
   **unconditionally** — regardless of whether the "intended" enforcement mechanism (an extension, a
   hook) is confirmed active. Never rely on a single point of enforcement for a hard safety
   constraint.
2. **An independent, hard server-side generation-length ceiling** (most llama.cpp-family servers
   expose a `-n` / `--n-predict`-style flag; the equivalent exists on other serving stacks) as a
   second, independent backstop — so even a client that forgets to cap `max_tokens` for some reason
   still can't run a generation to catastrophic length.
3. **Apply the full recommended sampler profile for the model class**, not just the headline
   repeat-penalty value — presence/frequency penalties and the DRY family are cheap to set and were
   specifically called out by the model vendor as assisting this exact failure mode.
4. **Verify with real end-to-end generations, not synthetic smoke tests**, across every
   box/backend actually serving the model. A single curl-style single-turn smoke test can look
   completely clean while the real multi-tool, real-prompt-size invocation still degenerates — the
   same load-dependence lesson as the [model-matrix doc](model-matrix.md#load-dependence-an-important-nuance)
   in Part 1: measure the shape of load you actually run, not a simplified proxy for it.

The general lesson, independent of this specific model family: **"there's a mechanism intended to
enforce X" is not evidence that X is actually enforced.** Verify liveness of every safety mechanism
independently, and apply redundant, independent enforcement for anything where an uncapped failure
is expensive (a crashed job, a corrupted downstream artifact, a wedged server — see §4).

---

## 4. Per-backend sampler-capability guard — a crash, not just a warning

### The incident

Once full sampler control is wired up (§2) and the "right" settings for a given model class are
known (§3), a further trap remains: **the same sampler parameter that is completely safe on one
serving backend can be unsafe, or outright fatal, on another** — and this is not a hypothetical
concern, it produced a real production outage.

A production elite-tier deployment — a large MoE model served via vLLM across a **multi-node,
tensor-parallel cluster with speculative decoding enabled** (multi-token-prediction-style draft
speculation for throughput) — received a routine sampler probe that included `repetition_penalty`.
The result was **not a clean HTTP error**: it triggered a `500` engine-core error that **wedged the
entire distributed serving collective**. The symptom was uniquely nasty: the server's own `/health`
endpoint kept responding `200 OK` — a naive liveness check would have reported the service as
healthy — while actual generation requests hung indefinitely. Recovery required a **coordinated
restart across every node in the cluster** (a single-node container restart cannot rejoin a peer's
stale distributed process group); a supervising watchdog process detected the wedge (because it
probed with **an actual one-token generation request, not just the health endpoint**) and performed
the coordinated multi-node restart automatically. Cold start after a wedge like this can run into
several minutes (model weight load + speculative-decoding draft-model load + kernel/graph
autotuning), which is the real cost of getting this wrong in production.

### The nuance that makes this hard to generalize from naively

The natural over-correction after an incident like this is "vLLM (or: this backend, or: speculative
decoding) doesn't support extra sampler parameters, avoid them all." **That conclusion is wrong, and
proving it wrong mattered**: probing the *same recovered server* afterward showed `min_p` was
accepted cleanly (`HTTP 200`, valid completion, server stayed healthy) — so the failure was not
categorical. It was **one specific parameter, on one specific engine build, in one specific
decoding-mode configuration** that was unsafe — not "vLLM rejects extra samplers" in general.
Separately, an unrelated family of parameters (the DRY-family repetition controls, which are a
llama.cpp-specific mechanism) were confirmed to be **silently ignored** by this backend rather than
erroring *or* crashing — a third distinct outcome. So across a small set of "extra" sampler fields
probed against one real deployment, the observed behaviors were: **clean crash, clean accept, and
silent no-op** — three different outcomes for parameters that all look equally "standard" from the
outside. **Sampler-parameter compatibility has to be established empirically, per (backend, engine
version, decoding-mode configuration) — it cannot be assumed safe, and it cannot be assumed uniform
even within one backend.**

### The fix — a per-provider sampler-capability guard, not disabling the feature that caused it

The tempting "safe" response — turn off speculative decoding fleet-wide so every sampler parameter
is safe to send — was explicitly rejected: speculative decoding was a deliberate, measured
throughput win on that deployment, and disabling it everywhere to permit a rarely-used parameter is
a bad trade, especially since the crash was traceable to one specific parameter, not the feature
itself.

Instead: **a declared per-provider/per-engine sampler allow/deny list, applied as a filter on the
fully-resolved sampler settings, immediately before they reach the transport that sends them** —
drop the unsupported/unsafe keys, log that they were dropped, and never raise an error or block the
job over it.

```
resolved_sampler = merge(model_class_defaults, user_overrides)   # e.g. the Class-3 profile from §3
                              │
                              ▼
              per-provider capability filter (keyed on the SPECIFIC deployment identity,
              not a generic "backend type" — a different deployment of the same backend
              software, without speculative decoding, might have zero problem with the
              same parameter)
                              │
                              ▼
                    filtered_sampler → sent to the model
```

Design properties worth calling out explicitly, because they're what make this safe to introduce
into an existing system without a big-bang migration:

- **Default is an empty blocklist — byte-identical to today when a provider isn't specifically
  declared.** Adding the guard changes nothing until a specific deployment is known to need
  filtering; this makes it safe to add broadly and only tighten where evidence exists.
  Backends/models that never touch the risky parameter (because their own defaults never set it)
  are unaffected regardless.
- **Key on deployment identity, not generic backend type.** `"this specific vLLM deployment with
  speculative decoding enabled"` and `"a different vLLM deployment without it"` are different
  capability profiles even though they're "the same backend" in the abstract — don't over-generalize
  a blocklist derived from one deployment's crash onto every deployment that happens to share
  its serving software.
- **Drop-and-log, never error, never crash the caller's job over it.** The point is that a job with
  a user-set override that happens to be unsafe for its target should still complete successfully
  (with the unsafe parameter silently omitted and logged), not fail the whole job — the job's
  intent (its other, safe settings) should still go through.
- **Guard every transport path that can reach the model, not just the primary one.** If your system
  has more than one way to reach a given backend (say, a CLI-harness path used for most traffic, and
  a direct HTTP/SDK path used for a different subset of jobs), a guard added to only one path leaves
  the other fully exposed to the same crash — this has to be applied at the point right before
  *any* transport sends the request, symmetrically.
- **Treat the literal allow/deny map as a pragmatic starting point, with an explicit upgrade path**
  toward a real capability-registry that your routing/dispatch layer probably already needs for
  other reasons (e.g., knowing which backends can serve which context lengths). A hardcoded rule
  map is a fine, honest v1; design the call sites so that map can be swapped for a registry lookup
  later without changing anything that calls them.

### The broader lesson on liveness checks

Worth generalizing past sampler params specifically: for any distributed or stateful serving backend
where a "wedge" is possible (the process is alive and answering shallow health checks, but not
actually doing useful work), **your liveness probe has to exercise the real capability, not just
ping an endpoint.** A `/health` check that only confirms the HTTP server is accepting connections
will happily report "healthy" through an entire wedged outage. A probe that performs a real,
minimal generation request is the only thing that actually validates the thing you care about — and
it's what allowed this incident to self-heal automatically instead of requiring a human to notice
and intervene.

---

## Practitioner takeaways

1. **An extension/hook mechanism is only as good as its activation guarantee.** If activation
   depends on per-host installed state, verify it's actually installed everywhere you think it is —
   don't trust "the code that loads it exists." Prefer ship-with-engine + explicit-path-load over
   any global discovery mechanism.
2. **Don't assume a harness forwards a sampler field just because the underlying API supports it.**
   Capture and inspect a real outgoing request. Where the harness can't carry what you need, inject
   below it via a before-request hook, driven by data the calling/orchestration layer controls (e.g.
   an environment variable), not by the harness's own limited schema.
3. **Uncapped generation is a production incident waiting to happen, especially on
   abliterated/uncensored creative finetunes** — apply the model class's full recommended sampler
   profile *and* an unconditional client-side length cap *and* an independent server-side length
   ceiling. Defense in depth, because "there's a mechanism for this" is not the same as "the
   mechanism is confirmed active."
4. **Sampler-parameter safety is per (backend, version, decoding-mode), not per-backend-in-general**
   — the same backend can crash, silently ignore, or cleanly accept different parameters
   simultaneously. Guard with a drop-and-log filter applied at every transport that can reach the
   model, default-empty so it's safe to introduce broadly, keyed on the specific deployment rather
   than a generic backend-type label.
5. **For any backend where a "wedge" (alive but non-functional) is possible, your health check has
   to actually exercise generation**, not just confirm the process is answering HTTP requests.

*This is a sanitized field report; empirical numbers and configuration details are from real
incidents on the model and hardware classes named above. Verify against your own build, backend
version, and deployment topology — this document tells you what to check and why, not a universal
constant to copy.*
