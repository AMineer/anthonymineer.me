---
title: "The API in Front of the AI"
description: "Build a fully local LLM gateway lab on your Mac with Bifrost and Ollama qwen3.5. Zero cloud, zero cost, zero copy-paste."
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

## You've Got APIs. Now You've Got AI APIs. Now What?

Picture this: you grab an Ollama model, wire it into your app locally — done. Celebrate. But two months later? You've got five apps, a handful of models, and absolutely no visibility into what's being called, how often, or what it's costing you in compute. Sound familiar?

Welcome to the reason LLM gateways exist. This is Part 1 of a two-part Field Notes series. Here we cover what an LLM gateway is, why you'd want one, and how to get Bifrost running locally on your Mac against Ollama with qwen3.5 — fully offline, fully free, fully yours.

---

## So… What Even Is an LLM Gateway?

Think of an LLM gateway as the **air traffic control tower** for your AI requests.

Every time your app wants to talk to a language model, that request flies through the gateway first. The gateway decides where to send it, who is allowed to send it, how much it costs, and whether it should be cached, retried, or blocked outright.

**Non-techy version:** it's a smart switchboard that sits between your apps and every AI model on your machine (or the planet), speaking everyone's language and keeping receipts.

**Techy version:** it's an OpenAI-compatible reverse proxy that normalizes request and response formats across providers, enforcing auth, rate limits, routing logic, cost tracking, and observability — all in one place.

---

## Why Do You Need One?

### 🔑 One virtual key to rule them all

Without a gateway, every app talks directly to every model. With a gateway, your apps get virtual keys from the gateway itself. Rotate or revoke in one place — every downstream app is covered.

### 🔀 Provider flexibility without code changes

Want to swap from qwen3.5 to llama3.2 for a specific workload? That's a config change, not a code deployment. Your app cannot tell the difference.

### 💸 Spend and usage tracking

Even with local models you want to know which apps are hammering your GPU, how many tokens are flowing, and which requests are slowest. Gateways give you that visibility out of the box.

### 🔄 Automatic fallback and load balancing

What happens when your primary model is busy or unavailable? Define a fallback chain: try qwen3.5 → if that fails, try llama3.2. The gateway handles it automatically — no retry logic in your app.

### 💬 Semantic caching

If ten requests ask essentially the same question within a short window, should you run the model ten times? Semantic caching returns the cached response instantly for similar queries, slashing redundant compute.

### 🔭 Observability

Logs, latency per model, request counts, error rates — all the stuff you'll eventually want a dashboard for. Gateways plug this in as middleware so you don't have to instrument every app.

---

## Why Bifrost for a Local Lab?

There are several solid open-source LLM gateways out there. For a local lab environment, Bifrost hits the sweet spot:

- **Zero-config startup.** One `npx` command and you have a running gateway with a web UI. No YAML required to get started.
- **Built in Go.** Lightweight, fast, and doesn't need a Python environment or virtual env to manage.
- **Web UI included.** Add providers, configure keys, and watch requests flow through a dashboard at `localhost:8080`.
- **OpenAI-compatible.** Any SDK or tool that works with OpenAI works with Bifrost. One line change in your app.
- **Ollama support.** Bifrost treats Ollama as a first-class provider. Point it at `localhost:11434` and you're done.
- **Free and open source.** Apache 2.0 license. Self-host forever at no cost.

---

## Prerequisites

Before we start, make sure you have the following on your Mac:

| Tool | Version | Install |
|---|---|---|
| Node.js | 18+ | `brew install node` |
| Ollama | Latest | `brew install ollama` |
| qwen3.5:latest | 6.6 GB | `ollama pull qwen3.5:latest` |
| Docker (optional) | Latest | [docs.docker.com](https://docs.docker.com) |

Verify everything is ready:

```bash
node --version       # should show v18 or higher
ollama list          # should show qwen3.5:latest
ollama serve         # start Ollama if not already running
```

---

## Step 1: Verify Ollama is Running

Before adding a gateway, confirm qwen3.5 responds on its own:

```bash
curl http://localhost:11434/api/chat \
  -d '{
    "model": "qwen3.5:latest",
    "messages": [{"role": "user", "content": "Say hello in one sentence."}],
    "stream": false
  }'
```

You should get a JSON response with the model's reply. If not, run `ollama serve` in a terminal first.

---

## Step 2: Start Bifrost

### Option A — npx (fastest)

Open a new terminal window and run:

```bash
npx -y @maximhq/bifrost
```

Bifrost downloads a pre-compiled Go binary for your architecture (arm64 for M-series Macs) and starts immediately. You should see:

```
Bifrost HTTP gateway starting...
Web UI available at http://localhost:8080
```

> **Tip:** The first npx run may take 15–30 seconds while the binary downloads. Subsequent starts are instant.

### Option B — Docker (with data persistence)

If you prefer Docker and want your configuration to survive restarts:

```bash
docker run -p 8080:8080 \
  -v $(pwd)/data:/app/data \
  maximhq/bifrost
```

The `-v $(pwd)/data:/app/data` flag persists your provider config and request logs to a local `./data` folder.

---

## Step 3: Add Ollama as a Provider

Open **http://localhost:8080** in your browser. Bifrost starts in UI-config mode — everything is configured through the web interface.

In the Bifrost dashboard:

1. Go to **Providers** in the sidebar
2. Click **Add Provider**
3. Select **Ollama** from the provider list
4. Set the base URL to `http://localhost:11434`
5. Save — no API key needed for local Ollama

Alternatively, register it via the API:

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

## Step 4: Send Your First Request Through the Gateway

With Bifrost running and Ollama registered, send a request through the gateway:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [{"role": "user", "content": "What is an LLM gateway? One paragraph."}]
  }'
```

You should get a full OpenAI-format response from qwen3.5, routed through Bifrost. Check the Bifrost dashboard — you'll see the request logged with latency, token count, and model used.

> ✅ **You did it.** Your entire AI stack is now running locally. No cloud. No API keys. No bill at the end of the month.

---

## Step 5: Run the Test Suite

Here are the key tests to validate your setup end-to-end.

### Smoke test

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [{"role": "user", "content": "Reply with just the word PONG"}]
  }'
```

### Streaming test

Tokens should arrive word-by-word in your terminal:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [{"role": "user", "content": "Count from 1 to 5, one number per line."}],
    "stream": true
  }'
```

### System prompt test

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

### Tool calling test

qwen3.5 supports native tool calling. This verifies Bifrost passes tool schemas and the model returns structured output:

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

### Concurrent load test

Fires 10 parallel requests — check the Bifrost dashboard afterward for latency stats:

```bash
for i in {1..10}; do
  curl -s -X POST http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"ollama/qwen3.5:latest\",\"messages\":[{\"role\":\"user\",\"content\":\"What is $i + $i?\"}]}" &
done
wait && echo "All 10 requests completed"
```

### Python SDK drop-in test

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

## Test Reference

| Test | What it validates |
|---|---|
| Smoke test | Gateway ↔ Ollama routing works |
| Streaming | SSE passthrough works correctly |
| System prompt | System role forwarded correctly |
| Tool calling | Structured function output works |
| Concurrent load | No dropped requests under parallel load |
| Python SDK | Drop-in OpenAI replacement works end-to-end |

---

## What's Next

You now have a fully functional local LLM gateway running on your Mac. qwen3.5 is serving requests through Bifrost with observability, routing, and zero cloud dependency.

In **Part 2**, we'll go deeper: building a local MCP server that exposes your own machine's tools — system info, shell commands, custom APIs — and wiring it through Bifrost so qwen3.5 can actually *do things* on your behalf. That's where it gets really interesting.

Until then, some things worth exploring in your new setup:

- Add a second local model (`ollama pull llama3.2`) and configure a fallback chain between them
- Set a virtual key with a token budget and watch the gateway enforce it
- Enable semantic caching in the Bifrost UI and run the same prompt twice — watch the second one come back instantly
- Check `http://localhost:8080` after your test suite — the dashboard logs every request with latency and token counts

---

*Anthony Mineer is a Senior Manager of Cloud Engineering writing about cloud infrastructure, AI, and platform engineering at [anthonymineer.me](https://anthonymineer.me). Part 2: MCP Gateway Setup coming soon.*
