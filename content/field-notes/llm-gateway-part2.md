---
title: "The API in Front of the AI: Part 2"
description: "Build a local MCP server in Node.js, wire it into Bifrost, and let qwen3.5 use your own machine's tools — all offline, all free."
date: 2026-03-24
tags: ["ai", "llm", "mcp", "bifrost", "ollama", "local-lab", "cloud-engineering"]
series: ["LLM Gateways"]
series_order: 2
showToc: true
TocOpen: false
draft: true
---

*Filed under: Cloud Engineering · AI Infrastructure · Local Lab*

---

## Picking Up Where We Left Off

In [Part 1](/field-notes/llm-gateway-part1/), we got Bifrost running locally on your Mac, wired up Ollama with qwen3.5, and verified the whole stack was humming. Requests flowing through the gateway, streaming working, tool calling confirmed.

Now we're going deeper.

This post is about MCP — the Model Context Protocol. If Part 1 was about giving your AI a reliable phone line, Part 2 is about giving it hands. By the end you'll have a local MCP server running on your machine that exposes real tools, connected through Bifrost, so qwen3.5 can actually *do things* on your behalf — check system info, run safe shell commands, do math — all without touching the cloud.

---

## What Is MCP?

MCP stands for Model Context Protocol. It's an open standard, originally released by Anthropic in late 2024 and now governed by the Linux Foundation, that defines how AI models discover and call external tools and data sources.

Before MCP, every AI framework had its own tool integration format. OpenAI's function calling, LangChain's tool abstraction, Anthropic's tool use spec — they all solved the same problem differently. If you wanted your database lookup to work with multiple AI clients, you wrote multiple integrations.

MCP fixes that. Write one server, and any MCP-compatible client can use it — Claude Desktop, your own agent, Bifrost, whatever. It's the USB-C moment for AI tooling.

There are two transport modes worth knowing:

- **stdio** — local, process-to-process communication via stdin/stdout. The gateway spawns your script as a subprocess. No ports, no networking, no auth needed. This is what we're building today.
- **HTTP/SSE** — remote, for servers others connect to over a network. That's a future post.

---

## The Architecture

The flow looks like this:

```
Your app → Bifrost (port 8080) → spawns MCP server via stdio → tool runs locally → result returned
```

Bifrost acts as both the LLM gateway *and* the MCP client. It spawns your local MCP server process, hands it tool requests from qwen3.5, and passes results back. Your app never needs to know any of this is happening — it just sends a chat request and gets a response back that includes tool results.

---

## Prerequisites

You need everything from Part 1 still running:

- Node.js 18+ (`node --version`)
- Ollama running with qwen3.5 (`ollama serve`)
- Bifrost running (`npx -y @maximhq/bifrost`)

Plus one new package — the official MCP SDK:

```bash
mkdir ~/my-local-mcp && cd ~/my-local-mcp
npm init -y
npm install @modelcontextprotocol/sdk zod
```

Update `package.json` to add `"type": "module"` so we can use ES module syntax:

```json
{
  "name": "my-local-mcp",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.11.0",
    "zod": "^3.24.4"
  }
}
```

---

## Step 1: Build the MCP Server

Create `server.js` in your `~/my-local-mcp/` folder:

```js
#!/usr/bin/env node
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { execSync } from "child_process";
import os from "os";

const server = new McpServer({
  name: "local-host-tools",
  version: "1.0.0",
});

// Tool 1: system info
server.tool(
  "get_system_info",
  "Returns CPU, memory, and OS info from the local machine",
  {},
  async () => {
    const info = {
      platform: os.platform(),
      arch: os.arch(),
      hostname: os.hostname(),
      totalMemoryGB: (os.totalmem() / 1e9).toFixed(2),
      freeMemoryGB: (os.freemem() / 1e9).toFixed(2),
      cpus: os.cpus().length,
      uptime_hours: (os.uptime() / 3600).toFixed(1),
    };
    return {
      content: [{ type: "text", text: JSON.stringify(info, null, 2) }],
    };
  }
);

// Tool 2: safe shell commands (allowlisted only)
server.tool(
  "run_command",
  "Runs a read-only shell command and returns output",
  { command: z.string().describe("The shell command to run") },
  async ({ command }) => {
    const allowed = ["df", "du", "ls", "pwd", "whoami", "date", "uptime", "ollama"];
    const base = command.trim().split(" ")[0];
    if (!allowed.includes(base)) {
      return {
        content: [{ type: "text", text: `Command '${base}' not in allowlist.` }],
        isError: true,
      };
    }
    try {
      const output = execSync(command, { encoding: "utf8", timeout: 5000 });
      return { content: [{ type: "text", text: output }] };
    } catch (err) {
      return { content: [{ type: "text", text: `Error: ${err.message}` }], isError: true };
    }
  }
);

// Tool 3: calculator
server.tool(
  "calculator",
  "Performs basic arithmetic",
  {
    operation: z.enum(["add", "subtract", "multiply", "divide"]),
    a: z.number(),
    b: z.number(),
  },
  async ({ operation, a, b }) => {
    const results = {
      add: a + b,
      subtract: a - b,
      multiply: a * b,
      divide: b !== 0 ? a / b : "cannot divide by zero"
    };
    return { content: [{ type: "text", text: String(results[operation]) }] };
  }
);

// ⚠️ Critical: NEVER use console.log() in a stdio MCP server
// It writes to stdout and corrupts the JSON-RPC stream
// Always use console.error() for any debug output
const transport = new StdioServerTransport();
await server.connect(transport);
console.error("MCP server running on stdio");
```

Three tools exposed:
- `get_system_info` — reads OS, CPU, memory stats from your machine
- `run_command` — runs safe, allowlisted shell commands
- `calculator` — basic arithmetic

> **The one rule you cannot break with stdio servers:** never use `console.log()`. It writes to stdout, which is the same pipe the MCP protocol uses for JSON-RPC messages. One stray `console.log()` will corrupt the stream and silently break everything. Use `console.error()` for all debug output — it goes to stderr instead.

---

## Step 2: Test the Server in Isolation

Before connecting it to Bifrost, verify it works on its own using the MCP Inspector:

```bash
cd ~/my-local-mcp
npx @modelcontextprotocol/inspector node server.js
```

This opens a browser UI at `localhost:5173` where you can call each tool interactively and see the raw JSON responses. If `get_system_info` returns your machine's stats and `calculator` returns correct math — your server is good.

---

## Step 3: Register the MCP Server in Bifrost

Make sure Bifrost is running, then register your local server via the API:

```bash
curl -X POST http://localhost:8080/api/mcp/client \
  -H "Content-Type: application/json" \
  -d '{
    "name": "local-host-tools",
    "connection_type": "stdio",
    "stdio_config": {
      "command": ["node", "/Users/YOUR_USERNAME/my-local-mcp/server.js"],
      "args": []
    }
  }'
```

Replace `/Users/YOUR_USERNAME/my-local-mcp/server.js` with the actual absolute path to your file. You can get it by running `pwd` inside the `my-local-mcp` folder.

Alternatively, add it through the Bifrost web UI at `http://localhost:8080`:

1. Go to **MCP Clients** in the sidebar
2. Click **Add MCP Client**
3. Set connection type to **stdio**
4. Set command to `node` and args to the full path of your `server.js`

Verify Bifrost picked it up:

```bash
curl http://localhost:8080/api/mcp/clients
```

You should see `local-host-tools` in the response.

---

## Step 4: Let qwen3.5 Use Your Tools

Now the fun part. Send a request that lets qwen3.5 reach for your local tools:

### Ask about your machine

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [
      {"role": "user", "content": "What are the system specs on this machine? Use the get_system_info tool."}
    ],
    "mcp_servers": ["local-host-tools"]
  }'
```

### Run a shell command

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [
      {"role": "user", "content": "How much disk space is available? Use the run_command tool with df -h."}
    ],
    "mcp_servers": ["local-host-tools"]
  }'
```

### Do some math

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "ollama/qwen3.5:latest",
    "messages": [
      {"role": "user", "content": "What is 1,847 multiplied by 93? Use the calculator tool."}
    ],
    "mcp_servers": ["local-host-tools"]
  }'
```

qwen3.5 will reason about which tool to use, call it through Bifrost, get the result, and incorporate it into its response — all running completely locally, no internet required.

---

## What Just Happened?

Let's be clear about what's actually going on here because it's worth appreciating:

1. Your app sent a plain chat request to Bifrost
2. Bifrost forwarded it to qwen3.5 via Ollama with the available tool schemas attached
3. qwen3.5 decided which tool to call and with what arguments
4. Bifrost spawned your `server.js` subprocess and sent it the tool call over stdio
5. Your server ran the tool, returned the result
6. Bifrost handed the result back to qwen3.5
7. qwen3.5 incorporated the result into a natural language response
8. Your app got back a normal chat response

Every step of that happened on your laptop. No cloud. No API keys. No data leaving your machine.

---

## Extending Your MCP Server

This is where it gets interesting for real use cases. The pattern scales to anything you can write a function for. A few ideas that make sense for a home lab:

**Check which Ollama models are loaded:**

```js
server.tool(
  "list_ollama_models",
  "Lists all locally available Ollama models",
  {},
  async () => {
    const output = execSync("ollama list", { encoding: "utf8" });
    return { content: [{ type: "text", text: output }] };
  }
);
```

**Read a local file:**

```js
server.tool(
  "read_file",
  "Reads the contents of a local text file",
  { path: z.string().describe("Absolute path to the file") },
  async ({ path }) => {
    const content = fs.readFileSync(path, "utf8");
    return { content: [{ type: "text", text: content }] };
  }
);
```

**Hit a local API or service:**

```js
server.tool(
  "check_bifrost_health",
  "Checks if Bifrost gateway is healthy",
  {},
  async () => {
    const res = await fetch("http://localhost:8080/health");
    const text = await res.text();
    return { content: [{ type: "text", text: text }] };
  }
);
```

The sky's the limit. Any tool you can wrap in a function, you can expose to your local AI.

---

## What's Next

We've now built a complete local AI stack from scratch:

- **Bifrost** as the gateway — routing, observability, fallbacks
- **Ollama + qwen3.5** as the model — offline, free, surprisingly capable
- **MCP** as the tool layer — giving the model real access to your environment

The natural next steps from here are building HTTP-based MCP servers (so tools can be shared across machines or teams), exploring multi-agent patterns where one model orchestrates others, and looking at how this same architecture maps to production — because the patterns are identical, just at a different scale.

More field notes coming. In the meantime — go build something and break things.

---

*Anthony Mineer is a Senior Manager of Cloud Engineering writing about cloud infrastructure, AI, and platform engineering at [anthonymineer.me](https://anthonymineer.me).*
