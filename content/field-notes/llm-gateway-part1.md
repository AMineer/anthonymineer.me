---
title: "The API in Front of the AI"
description: "Standing up a local LLM gateway with Bifrost and Ollama qwen3.5. No cloud, no keys, no bill."
date: 2026-03-24
tags: ["ai", "llm", "bifrost", "ollama", "local-lab", "cloud-engineering"]
series: ["LLM Gateways"]
series_order: 1
showToc: true
TocOpen: false
cover:
  alt: "LLM Gateway reference architecture diagram"
draft: false
---

*Filed under: Cloud Engineering · AI Infrastructure · Local Lab*

---

## The problem

You wire an Ollama model into an app and it works. A couple months later you've got five apps, a few models, and no idea what's calling what, how often, or what it's doing to your GPU.

That's the gap an LLM gateway fills. This is Part 1 of a two-part series. Here we cover what a gateway does, why it's worth running, and how to get Bifrost talking to Ollama qwen3.5 on your Mac. Fully local.

---

## What a gateway actually does

It's an OpenAI-compatible reverse proxy that sits between your apps and your models. Every request goes through it first. It normalizes request and response formats across providers and gives you one place to handle auth, routing, rate limits, cost tracking, and logging.

Concretely, that buys you:

- **One key surface.** Apps get virtual keys from the gateway instead of talking to each model directly. Rotate or revoke in one place.
- **Provider swaps without redeploys.** Moving a workload from qwen3.5 to llama3.2 is a config change. The app doesn't know.
- **Usage visibility.** Token counts, request volume, and latency per model, even for local models hammering your hardware.
- **Fallback chains.** Primary busy or down? Define qwen3.5 then llama3.2 and let the gateway handle the retry. No logic in your app.
- **Semantic caching.** Near-identical prompts return a cached response instead of re-running the model.
- **Observability as middleware.** Logs, latency, error rates without instrumenting every app.

---

## Why Bifrost

A few open-source gateways do this well. Bifrost fits a local lab for a few reasons:

- One `npx` command and you have a running gateway with a web UI. No YAML to start.
- Written in Go, so no Python env to babysit.
- Treats Ollama as a first-class provider. Point it at `localhost:11434` and you're done.
- OpenAI-compatible, so any SDK that talks to OpenAI talks to Bifrost with a base URL change.
- Apache 2.0. Self-host for free.

---

## Prerequisites

| Tool | Version | Install |
|---|---|---|
| Node.js | 18+ | `brew install node` |
| Ollama | Latest | `brew install ollama` |
| qwen3.5:latest | 6.6 GB | `ollama pull qwen3.5:latest` |
| Docker (optional) | Latest | [docs.docker.com](https://docs.docker.com) |

Confirm the basics:

```bash
node --version       # v18+
ollama list          # should list qwen3.5:latest
ollama serve         # start it if it isn't running
```

---

## Step 1: Confirm Ollama responds

Before adding anything in front of it, make sure qwen3.5 answers on its own:

```bash
curl http://localhost:11434/api/chat \
  -d '{
    "model": "qwen3.5:latest",
    "messages": [{"role": "user", "content": "Say hello in one sentence."}],
    "stream": false
  }'
```

If you get JSON back, you're good. If not, run `ollama serve` first.

---

## Step 2: Start Bifrost

### npx

```bash
npx -y @maximhq/bifrost
```

Bifrost pulls a pre-compiled binary for your architecture (arm64 on M-series) and starts:

```
Bifrost HTTP gateway starting...
Web UI available at http://localhost:8080
```

First run takes 15 to 30 seconds for the download. After that it's instant.

### Docker (if you want config to persist)

```bash
docker run -p 8080:8080 \
  -v $(pwd)/data:/app/data \
  maximhq/bifrost
```

The volume mount persists provider config and request logs to `./data`.

---

## Step 3: Register Ollama

Open **http://localhost:8080**. Bifrost starts in UI-config mode.

In the dashboard: **Providers → Add Provider → Ollama**, set the base URL to `http://localhost:11434`, save. No key needed for local.

Or via the API:

```bash
curl -X POST http://localhost:8080/api/providers \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "ollama",
    "base_url": "http://localhost:11434",
    "keys": [{"name": "local", "value": "none", "models": ["qwen3.5:latest"]}]
  }'
```

---

## Step 4: First request through the gateway

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [{"role": "user", "content": "What is an LLM gateway? One paragraph."}]
  }'
```

You get a standard OpenAI-format response, routed through Bifrost. The request shows up in the dashboard with latency, token count, and model.

---

## Step 5: Validate it end to end

### Smoke

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [{"role": "user", "content": "Reply with just the word PONG"}]
  }'
```

### Streaming

Tokens should arrive incrementally:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [{"role": "user", "content": "Count from 1 to 5, one number per line."}],
    "stream": true
  }'
```

### System prompt

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [
      {"role": "system", "content": "You are a pirate. Respond only in pirate speak."},
      {"role": "user", "content": "Explain what a cloud gateway is."}
    ]
  }'
```

### Tool calling

qwen3.5 supports native tool calling. This checks that Bifrost passes the schema through and the model returns structured output:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [{"role": "user", "content": "What is 47 multiplied by 13? Use the calculator."}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "calculator",
        "description": "Performs arithmetic",
        "parameters": {
          "type": "object",
          "properties": {
            "operation": {"type": "string", "enum": ["add","subtract","multiply","divide"]},
            "a": {"type": "number"},
            "b": {"type": "number"}
          },
          "required": ["operation","a","b"]
        }
      }
    }],
    "tool_choice": "auto"
  }'
```

### Concurrent load

Ten parallel requests. Check the dashboard afterward for latency under load:

```bash
for i in {1..10}; do
  curl -s -X POST http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"ollama/qwen3.5:latest\",\"messages\":[{\"role\":\"user\",\"content\":\"What is $i + $i?\"}]}" &
done
wait && echo "All 10 requests completed"
```

### Python SDK drop-in

```python
import openai

client = openai.OpenAI(
    api_key="dummy",
    base_url="http://localhost:8080/openai"
)

response = client.chat.completions.create(
    model="ollama/qwen3.5:latest",
    messages=[{"role": "user", "content": "Explain an LLM gateway in one sentence."}]
)
print(response.choices[0].message.content)
print(f"Model: {response.model}")
print(f"Tokens: {response.usage.total_tokens}")
```

---

## Test reference

| Test | Validates |
|---|---|
| Smoke | Gateway to Ollama routing |
| Streaming | SSE passthrough |
| System prompt | System role forwarded |
| Tool calling | Structured function output |
| Concurrent load | No drops under parallel load |
| Python SDK | Drop-in OpenAI replacement |

---

## Next

You've got a working local gateway. qwen3.5 is serving through Bifrost with logging, routing, and no cloud dependency.

Part 2 goes further: a local MCP server exposing your own machine's tools (system info, shell, custom APIs) wired through Bifrost so qwen3.5 can actually run things for you.

Worth poking at before then:

- Pull a second model (`ollama pull llama3.2`) and set a fallback chain between the two
- Give a virtual key a token budget and watch the gateway enforce it
- Turn on semantic caching and run the same prompt twice
- Check the dashboard after the test suite for per-request latency and token counts

---

*[anthonymineer.me](https://anthonymineer.me). Part 2: llm-gateway-part2.*