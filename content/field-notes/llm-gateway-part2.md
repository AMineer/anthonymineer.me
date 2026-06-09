---
title: "The API in Front of the AI: Part 2"
description: "A local MCP server in Node.js, wired into Bifrost, giving qwen3.5 access to your own machine's tools. Offline and free."
date: 2026-03-31
tags: ["ai", "llm", "mcp", "bifrost", "ollama", "local-lab", "cloud-engineering"]
series: ["LLM Gateways"]
series_order: 2
showToc: true
TocOpen: false
draft: false
---

*Filed under: Cloud Engineering · AI Infrastructure · Local Lab*

---

## Where Part 1 left off

In [Part 1](/field-notes/llm-gateway-part1/) we got Bifrost running locally, wired up Ollama with qwen3.5, and confirmed the stack end to end. Requests through the gateway, streaming, tool calling.

This post adds MCP, the Model Context Protocol. Part 1 gave the model a reliable connection. This part gives it tools. By the end you'll have a local MCP server exposing real capabilities (system info, allowlisted shell commands, math) connected through Bifrost so qwen3.5 can run them. Still no cloud.

---

## What MCP is

Model Context Protocol is an open standard, released by Anthropic in late 2024 and now under the Linux Foundation, that defines how models discover and call external tools and data sources.

Before it, every framework had its own tool format. OpenAI function calling, LangChain's tool abstraction, Anthropic's tool use spec all solved the same problem differently. Supporting multiple clients meant writing multiple integrations.

MCP collapses that to one. Write a server once and any MCP-compatible client can use it: Claude Desktop, your own agent, Bifrost, anything.

Two transport modes matter here:

- **stdio.** Local, process-to-process over stdin/stdout. The gateway spawns your script as a subprocess. No ports, no networking, no auth. This is what we're building.
- **HTTP/SSE.** Remote, for servers others reach over a network. Another post.

---

## The architecture

```
Your app → Bifrost (port 8080) → spawns MCP server via stdio → tool runs locally → result returned
```

Bifrost is both the LLM gateway and the MCP client. It spawns your server process, hands it tool requests from qwen3.5, and passes results back. The app just sends a chat request and gets a response that includes the tool results.

---

## Prerequisites

Everything from Part 1 still running:

- Node.js 18+ (`node --version`)
- Ollama with qwen3.5 (`ollama serve`)
- Bifrost (`npx -y @maximhq/bifrost`)

Plus the MCP SDK:

```bash
mkdir ~/my-local-mcp && cd ~/my-local-mcp
npm init -y
npm install @modelcontextprotocol/sdk zod
```

Add `"type": "module"` to `package.json` for ES module syntax:

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

## Step 1: Build the server

Create `server.js` in `~/my-local-mcp/`:

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

// Critical: never use console.log() in a stdio MCP server.
// It writes to stdout, the same pipe the JSON-RPC stream uses, and corrupts it.
// Use console.error() for any debug output.
const transport = new StdioServerTransport();
await server.connect(transport);
console.error("MCP server running on stdio");
```

Three tools: `get_system_info` reads OS, CPU, and memory stats; `run_command` runs allowlisted shell commands; `calculator` does arithmetic.

> The one rule you can't break with stdio servers: never `console.log()`. It writes to stdout, which is the same pipe MCP uses for JSON-RPC. One stray log call corrupts the stream and breaks everything silently. Send all debug output to stderr with `console.error()`.

---

## Step 2: Test the server alone

Before touching Bifrost, verify the server works on its own with the MCP Inspector:

```bash
cd ~/my-local-mcp
npx @modelcontextprotocol/inspector node server.js
```

That opens a UI at `localhost:5173` where you can call each tool and inspect the raw JSON. If `get_system_info` returns your stats and `calculator` does the math, the server is good.

---

## Step 3: Register the server in Bifrost

With Bifrost running, register the server via the API:

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

Replace the path with your absolute path (run `pwd` in the `my-local-mcp` folder to get it).

Or use the web UI at `http://localhost:8080`:

1. **MCP Clients** in the sidebar
2. **Add MCP Client**
3. Connection type **stdio**
4. Command `node`, args set to the full path of `server.js`

Confirm Bifrost registered it:

```bash
curl http://localhost:8080/api/mcp/clients
```

You should see `local-host-tools` in the response.

---

## Step 4: Let qwen3.5 use the tools

Send requests that let the model reach for your local tools.

### Ask about the machine

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

qwen3.5 reasons about which tool to use, calls it through Bifrost, gets the result, and folds it into the response. All local.

---

## What happened on that last call

1. Your app sent a plain chat request to Bifrost
2. Bifrost forwarded it to qwen3.5 via Ollama with the tool schemas attached
3. qwen3.5 picked a tool and its arguments
4. Bifrost spawned `server.js` and sent it the tool call over stdio
5. The server ran the tool and returned the result
6. Bifrost handed the result back to qwen3.5
7. qwen3.5 wrote a natural language response around it
8. Your app got back a normal chat response

Every step ran on your laptop. Nothing left the machine.

---

## Extending the server

The pattern scales to anything you can write a function for. A few that earn their place in a home lab:

**List loaded Ollama models:**

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

**Read a local file** (add `import fs from "fs";` to the top of `server.js` alongside the other imports):

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

**Hit a local service:**

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

Anything you can wrap in a function, you can expose to the model.

---

## What's next

The full local stack is built:

- **Bifrost** for routing, observability, and fallbacks
- **Ollama + qwen3.5** as the model, offline and free
- **MCP** as the tool layer with real access to your environment

From here the natural directions are HTTP-based MCP servers so tools can be shared across machines, multi-agent patterns where one model orchestrates others, and mapping this same shape to production. The patterns hold at scale, the numbers just get bigger.

More notes coming.

---

*Anthony Mineer · [anthonymineer.me](https://anthonymineer.me)*