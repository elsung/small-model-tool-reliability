# Example grammars & usage

Illustrative GBNF grammars and a minimal calling pattern for the technique
described in the [main writeup](../README.md). These are **generic examples** —
adapt them to your own artifacts.

## Files

| file | what it forces | when to use |
|---|---|---|
| [`markdown-doc.gbnf`](./markdown-doc.gbnf) | output opens at a markdown heading, body free | any markdown artifact; the minimal anti-narration constraint |
| [`outline.gbnf`](./outline.gbnf) | a series of `## Episode N` sections, bodies free | a phase whose artifact has a known repeating structure |

## The two integration patterns

**(A) Direct call — recommended for single-output phases.** Call the server
directly with the phase's grammar and a bounded `max_tokens`, then write the
response to the target file. This bypasses the agent's tool loop entirely:
guaranteed structured, bounded, fast, ~0 narration.

Conceptually (any OpenAI-compatible llama.cpp server):

```jsonc
POST /v1/chat/completions
{
  "model": "gemma-4-12b-qat",
  "messages": [
    {"role": "system", "content": "Write the outline. Output only the artifact."},
    {"role": "user",   "content": "<the source brief, inlined>"}
  ],
  "grammar": "<contents of outline.gbnf>",   // llama.cpp-native grammar field
  "max_tokens": 4000                          // REQUIRED: cap open-ended grammars
}
```

Or force a JSON object via schema instead of a GBNF file:

```jsonc
{
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "artifact",
      "schema": {
        "type": "object",
        "properties": { "path": {"type": "string"}, "content": {"type": "string"} },
        "required": ["path", "content"]
      }
    }
  }
}
```

**(B) Via the agent harness — for keeping the tool loop.** If you must keep the
agent CLI in the loop, inject the `grammar` (and the `max_tokens` cap) into the
provider request for the designated single-output phase, and run the model with
tools disabled for that turn. The effect is the same: the model can only emit a
grammar-legal artifact.

## Cross-backend note

Both patterns are **sampler-level** and therefore work on serving backends where
the native templated-tool path is broken or disabled — see
[../docs/hardware-notes.md](../docs/hardware-notes.md).
