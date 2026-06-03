# Hermes 4.3 36B — Tools + Thinking (Q8_0)

A properly-configured Ollama packaging of **Nous Research's Hermes 4.3 36B**, built on the
**Q8_0** GGUF with the correct **Llama-3** chat template and **verified** `tools` + `thinking`
capabilities.

Maintained by **Monomyth Development**.

## What this fixes

Hermes 4.3 36B is fully tool-trained, but many community Ollama/GGUF uploads advertise only
`completion` ("Text") capability — agent frameworks that pass a `tools` array then error or
silently lose tool calling. The cause is the **Modelfile template**, not the weights: uploads
frequently ship a **ChatML** template (`<|im_start|>` / `<|im_end|>`), which is the Hermes-4
**14B** format. Hermes 4.3 **36B is Llama-3** (`<|start_header_id|>` / `<|eot_id|>`).

This build applies the correct Llama-3 template — adapted from
[`steelpuddles/hermes-4.3-36B:thinking-tools`](https://ollama.com/steelpuddles/hermes-4.3-36B),
who did the original template work — with the conditional structures Ollama's parser reads to
detect capabilities:

- **Tools** capability via the `.Tools` template branch
- **Thinking** capability, switched on the native `think` request parameter
- **Llama-3 chat template** (not ChatML), with `<tool_call>` / `<tool_response>` framing
- **Stop sequences** matched to model output (`<|eot_id|>`, `<|end_of_text|>`)
- **Default context** raised to 32K (the stock 4096 truncates tool definitions)
- **Q8_0 quant** — prioritizes quality over footprint

Confirm after pulling:

```bash
ollama show MonomythDevelopment/hermes-4.3-36b-tools
# Capabilities: completion, tools, thinking
```

## Quick start

```bash
ollama pull MonomythDevelopment/hermes-4.3-36b-tools
ollama run  MonomythDevelopment/hermes-4.3-36b-tools "What's 2+2?"
```

## Thinking (per-request toggle)

Thinking is a reasoning mode (the model emits `<think>…</think>` before answering), mapped to
Ollama's native `think` field — orthogonal to tool calling, controlled independently. Default it
off for agent loops (no `<think>` blocks to strip from tool-call output) and opt in where
deliberation helps.

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "MonomythDevelopment/hermes-4.3-36b-tools",
  "messages": [{"role": "user", "content": "Prove that sqrt(2) is irrational."}],
  "think": true,
  "stream": false
}'
```

## Tool calling

Tools are declared as OpenAI-style JSON schemas; the model emits
`<tool_call>{"name": …, "arguments": {…}}</tool_call>`; results return in
`<tool_response>…</tool_response>`. Any OpenAI-compatible client that sends a `tools` array
works — verify by confirming `message.tool_calls` is populated.

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "MonomythDevelopment/hermes-4.3-36b-tools",
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

**Runtime caveat:** seed_oss tool-call *parsing* in llama.cpp/Ollama is still maturing — a
correct template can't fix an immature parser underneath it. If tool calls parse unreliably,
update to the latest Ollama, or serve with **vLLM**
(`--enable-auto-tool-choice --tool-call-parser hermes`). Verify on your own build before relying
on it.

## Parameters

| Parameter | Value |
|-----------|-------|
| Quant | Q8_0 (~38 GB) |
| `num_ctx` | 32768 (native max 524288) |
| `temperature` | 0.6 |
| `top_p` | 0.95 |
| `top_k` | 20 |
| Stops | `<|eot_id|>`, `<|end_of_text|>` |

## License

Apache 2.0 throughout — commercial use permitted. Copyright 2026 Monomyth Development.

## Credits

This is a **packaging** of others' work; it adds no weights of its own.

- **ByteDance Seed Team** — base model ([Seed-OSS-36B-Base](https://huggingface.co/ByteDance-Seed/Seed-OSS-36B-Base))
- **Nous Research** — fine-tune and Q8_0 GGUF ([Hermes-4.3-36B](https://huggingface.co/NousResearch/Hermes-4.3-36B), [GGUF](https://huggingface.co/NousResearch/Hermes-4.3-36B-GGUF))
- **Steel Puddles** — original Llama-3 tools+thinking template ([steelpuddles/hermes-4.3-36B](https://ollama.com/steelpuddles/hermes-4.3-36B))

Source repo & full attribution: <https://github.com/MonomythDevelopment/ollama-hermes-4.3-36b-tools>
