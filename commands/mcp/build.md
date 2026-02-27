# /mcp:build

Scaffold a production-ready custom MCP server from a plain-English description of what tools to expose. No boilerplate hunting — design, generate, wire up, and document in one pass.

---

## Parse $ARGUMENTS

- **description** — what tools to expose, e.g. "query our internal REST API, cache responses in Redis" (required)
- `--name <server-name>` — server name in kebab-case (default: derived from description)
- `--lang <ts|py>` — implementation language (default: `ts`)
- `--out <path>` — output directory (default: `./mcp-servers/<name>`)
- `--setup` — automatically run `/mcp:setup` after scaffolding to wire into Claude settings

---

## Workflow

### 1 · Design Tools

Spawn an analysis agent — subagent_type: `Explore`, thoroughness: `very thorough`:

```
Analyze this request and design a precise MCP tool schema:

Request: <description>
Server name: <name>
Language: <lang>

For each tool, define:
- name: snake_case, verb-first (e.g. get_user, search_documents, create_ticket)
- description: one sentence, clear enough that an LLM knows when to call it
- inputSchema: full JSON Schema with required/optional params, types, descriptions
- returns: describe what the tool returns (string, JSON object, etc.)
- errors: what can go wrong, what error messages to surface

Also identify:
- Required env vars (API keys, connection strings, etc.)
- External dependencies (npm packages or pip packages needed)
- Any auth patterns (API key header, OAuth token, basic auth)
- Rate limiting or pagination concerns

Output a structured tool manifest — not code, just the design.
```

**APPROVAL GATE — present tool manifest. Use AskUserQuestion:**
> Approve these tools? → Approve / Revise / Cancel

Do not write a line of code until explicitly approved.

---

### 2 · Scaffold Server

Spawn an implementation agent — subagent_type: `general-purpose`, isolation: `worktree`:

```
Scaffold a production-ready MCP server with these approved tools:

<tool manifest>

Server name: <name>
Language: <lang>
Output directory: <out>

--- TYPESCRIPT STRUCTURE (if lang=ts) ---

<out>/
  src/
    index.ts          # main server — transport, tool registration, request handlers
    tools/
      <tool-name>.ts  # one file per tool — implementation + schema export
    types.ts          # shared TypeScript interfaces
    config.ts         # env var loading with validation (throw on missing required vars)
    errors.ts         # typed error classes
  tests/
    <tool-name>.test.ts  # unit test per tool (vitest)
  package.json
  tsconfig.json
  .env.example
  README.md

package.json must include:
  - "@modelcontextprotocol/sdk": "latest"
  - "zod" for input validation
  - "vitest" for tests
  - build script: "tsc"
  - start script: "node dist/index.js"

index.ts pattern:
  import { Server } from "@modelcontextprotocol/sdk/server/index.js"
  import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"
  import { ListToolsRequestSchema, CallToolRequestSchema } from "@modelcontextprotocol/sdk/types.js"

  const server = new Server({ name: "<name>", version: "0.1.0" }, { capabilities: { tools: {} } })
  server.setRequestHandler(ListToolsRequestSchema, async () => ({ tools: [...] }))
  server.setRequestHandler(CallToolRequestSchema, async (request) => { ... })
  const transport = new StdioServerTransport()
  await server.connect(transport)

Each tool file exports:
  - toolDefinition: { name, description, inputSchema }
  - handler(args): Promise<{ content: [{ type: "text", text: string }] }>

--- PYTHON STRUCTURE (if lang=py) ---

<out>/
  server.py           # main server
  tools/
    __init__.py
    <tool_name>.py    # one file per tool
  config.py           # env var loading
  errors.py           # typed exceptions
  tests/
    test_<tool_name>.py
  pyproject.toml
  .env.example
  README.md

pyproject.toml must include:
  - mcp >= 1.0.0
  - pydantic for input validation
  - pytest + pytest-asyncio for tests

server.py pattern:
  from mcp.server import Server
  from mcp.server.models import InitializationOptions
  import mcp.types as types
  import asyncio, mcp

  app = Server("<name>")

  @app.list_tools()
  async def handle_list_tools() -> list[types.Tool]: ...

  @app.call_tool()
  async def handle_call_tool(name: str, arguments: dict) -> list[types.TextContent]: ...

  async def main():
      async with mcp.server.stdio.stdio_server() as (read, write):
          await app.run(read, write, InitializationOptions(...))

  asyncio.run(main())

--- UNIVERSAL STANDARDS ---

- config.ts/config.py: load all env vars at startup, fail fast with clear error if required vars missing
- Every tool handler: validate inputs with Zod/Pydantic, return typed errors, never throw unhandled
- Error response format: { content: [{ type: "text", text: "Error: <message>" }], isError: true }
- .env.example: every env var with description and example value
- README.md: tool table, env vars table, install + run instructions, claude settings entry
```

Generate the Claude settings entry:

```json
{
  "<name>": {
    "command": "node",
    "args": ["<out>/dist/index.js"]
  }
}
```
(or `"python"` + `["<out>/server.py"]` for Python)

---

### 3 · Build & Smoke Test

```bash
# TypeScript
cd <out> && npm install && npm run build

# Python
cd <out> && pip install -e ".[dev]"
```

Spawn a test agent — subagent_type: `testing-suite:test-engineer`:

```
Run the test suite for this MCP server: <out>

Verify:
- All tool definitions are valid (name, description, inputSchema present)
- Each tool handler returns the correct shape: { content: [{ type, text }] }
- Error paths return isError: true
- Config validation throws on missing required env vars

Report: PASSED or FAILED with exact failures.
```

If FAILED: spawn a fix agent (subagent_type: `general-purpose`), fix, re-test once.

---

### 4 · Document

Verify `README.md` contains:

```markdown
# <name> MCP Server

## Tools
| Tool | Description | Required params |
|------|-------------|----------------|
| tool_name | ... | param1, param2 |

## Environment Variables
| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| API_KEY | Yes | ... | sk-... |

## Install & Run
npm install && npm run build

## Claude Settings Entry
\`\`\`json
{ "<name>": { "command": "node", "args": ["dist/index.js"] } }
\`\`\`

## Add to Claude
Run: /mcp:setup <name> --path <out>
```

---

### 5 · Output

Print:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ MCP server scaffolded: <name>
  Location:  <out>
  Language:  <lang>
  Tools:     <count> (<tool names>)
  Env vars:  <required vars>

  Next steps:
  1. Fill in tool implementations in <out>/src/tools/ (marked with TODO)
  2. Add env vars to <out>/.env
  3. Wire into Claude:
     /mcp:setup <name> --path <out>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If `--setup` flag was passed: automatically invoke `/mcp:setup <name> --path <out>`.
