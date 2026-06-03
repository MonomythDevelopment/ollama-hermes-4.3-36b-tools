# hermes-4.3-36b-tools

A tools- and thinking-enabled Ollama packaging of **NousResearch Hermes-4.3-36B**, built on
the **Q8_0** GGUF with the correct **Llama-3** chat template.

Maintained by **Monomyth Development**.

The Hermes-4.3-36B weights are already fully tool-trained, but many community Ollama/GGUF
uploads advertise only `completion` capability — agent frameworks that pass tool definitions
get 400s or silently lose tool calling. The cause is the **Modelfile template**, not the
weights: uploads frequently ship a **ChatML** template (`<|im_start|>` / `<|im_end|>`), which
is the Hermes-4 **14B** format. Hermes 4.3 **36B is Llama-3** (`<|start_header_id|>` /
`<|eot_id|>`). This build applies the correct Llama-3 template with the conditional structures
Ollama's parser reads to detect and advertise capabilities — flipping `ollama show` to report
`tools` and `thinking`.

## Quick start

```bash
# From the registry (once published):
ollama pull MonomythDevelopment/hermes-4.3-36b-tools

# Or build it yourself from this repo:
ollama create hermes-4.3-36b-tools -f Modelfile

# Confirm the capabilities flipped on:
ollama show hermes-4.3-36b-tools     # Capabilities: completion, tools, thinking
```

## Thinking is a per-request toggle, not a tool

"Thinking" is a reasoning mode (the model emits `<think>…</think>` before answering), gated
in-template by `{{ if and .IsThinkSet .Think }}`. It maps to Ollama's native `think` field and
is **orthogonal** to tool calling — you control them independently:

```bash
# Fast structured answer, no reasoning trace:
ollama run hermes-4.3-36b-tools "What's 2+2?"

# Deliberate, with an explicit <think> chain:
ollama run hermes-4.3-36b-tools --think "Prove that sqrt(2) is irrational."
```

For agent / decision-loop use, default `think` off (no `<think>` blocks to strip from tool-call
output) and opt in only where deliberation helps.

## Tool calling

Hermes uses the long-standing convention: tools are declared in the system prompt inside
`<tools>…</tools>` as OpenAI-style JSON schemas; the model emits calls as
`<tool_call>{"name": …, "arguments": {…}}</tool_call>`; results are returned in
`<tool_response>…</tool_response>`. Any OpenAI-compatible client that sends a `tools` array
works — verify by confirming `message.tool_calls` is populated:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "hermes-4.3-36b-tools",
  "messages": [{"role": "user", "content": "What is the weather in Paris?"}],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "Get current weather for a city",
      "parameters": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"]
      }
    }
  }],
  "stream": false
}'
```

## Background

- **Quant:** Q8_0 (~38 GB) — prioritizes quality; the model is small enough that an
  unconstrained machine should not drop below it. Lower quants (Q6_K ~30 GB, Q5_K_M ~26 GB,
  Q4_K_M ~22 GB) trade quality for footprint; swap the `FROM` tag to use them.
- **Context:** `num_ctx 32768`. The stock config ships at 4096, which truncates tool
  definitions and breaks agent loops. The native max is 524288, but that consumes most of the
  KV-cache budget — 32K is a practical default.
- **Sampling:** Nous non-reasoning defaults — `temperature 0.6`, `top_p 0.95`, `top_k 20`.
- **Runtime caveat (the real one):** seed_oss tool-call **parsing** in llama.cpp/Ollama is
  still maturing (a correct template cannot fix an immature parser underneath it). If tool
  calls parse unreliably, update to the latest Ollama build, or serve with **vLLM**
  (`--enable-auto-tool-choice --tool-call-parser hermes`), which has solid seed_oss/hermes
  tool parsing. Verify tool calling on your own build before relying on it.

## Attribution & lineage

This is a **packaging** of others' work. The template is adapted from
[`steelpuddles/hermes-4.3-36B:thinking-tools`](https://ollama.com/steelpuddles/hermes-4.3-36B)
(Apache-2.0). Full lineage:

- **Base model:** [ByteDance Seed-OSS-36B-Base](https://huggingface.co/ByteDance-Seed/Seed-OSS-36B-Base) (Apache-2.0)
- **Fine-tune:** [NousResearch/Hermes-4.3-36B](https://huggingface.co/NousResearch/Hermes-4.3-36B) (Apache-2.0)
- **GGUF quants:** [NousResearch/Hermes-4.3-36B-GGUF](https://huggingface.co/NousResearch/Hermes-4.3-36B-GGUF)
- **Template reference:** steelpuddles/hermes-4.3-36B:thinking-tools (Apache-2.0)

See [`NOTICE`](./NOTICE) for the attribution notice and [`LICENSE`](./LICENSE) for the full
Apache-2.0 text. This packaging adds no weights of its own — only the Modelfile, template
configuration, and documentation.

## Publishing (maintainers)

```bash
ollama create MonomythDevelopment/hermes-4.3-36b-tools -f Modelfile
ollama push   MonomythDevelopment/hermes-4.3-36b-tools
```

The Ollama registry holds the built, distributable artifact; this git repo is the
human-readable, versioned source of truth. Tag changelog entries to each registry push.

## License

Apache-2.0. Copyright 2026 Monomyth Development. See [`LICENSE`](./LICENSE).
