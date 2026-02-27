# /mcp:setup

Add, configure, and validate MCP servers in Claude settings. Handles well-known servers from the catalog, custom server paths, env var collection, connection testing, and tool documentation — in one pass.

---

## Parse $ARGUMENTS

- **server** — well-known server name OR path to custom server (required)
- `--scope <user|project>` — where to configure (default: `user` → `~/.claude/settings.json`, `project` → `.claude/settings.json`)
- `--env <KEY=VALUE,...>` — env vars to pass directly (skips interactive prompts for those keys)
- `--path <dir>` — path to local MCP server directory (for custom servers built with `/mcp:build`)
- `--name <alias>` — override the server key name in settings (default: server name)
- `--remove` — remove an existing MCP server from settings

---

## Well-Known Server Catalog

Recognize these server names and auto-resolve their config:

| Name | Package | Required env vars | Key tools |
|------|---------|-------------------|-----------|
| `playwright` | `@playwright/mcp@latest` | — | navigate, screenshot, click, fill, evaluate |
| `postgres` / `postgresql` | `@modelcontextprotocol/server-postgres` | `DATABASE_URL` | query, list_tables, describe_table |
| `github` | `@modelcontextprotocol/server-github` | `GITHUB_PERSONAL_ACCESS_TOKEN` | create_issue, get_file, create_pr, search_repos |
| `filesystem` | `@modelcontextprotocol/server-filesystem` | — | read_file, write_file, list_directory, move_file |
| `memory` | `@modelcontextprotocol/server-memory` | — | create_entities, add_observations, search_nodes, open_nodes |
| `sqlite` | `@modelcontextprotocol/server-sqlite` | — | query, list_tables, describe_table, create_table |
| `brave-search` | `@modelcontextprotocol/server-brave-search` | `BRAVE_API_KEY` | brave_web_search, brave_local_search |
| `slack` | `@modelcontextprotocol/server-slack` | `SLACK_BOT_TOKEN`, `SLACK_TEAM_ID` | list_channels, post_message, reply_to_thread, get_users |
| `fetch` | `@modelcontextprotocol/server-fetch` | — | fetch (retrieve any URL as text/markdown) |
| `context7` | `@context7/mcp` | — | resolve-library-id, get-library-docs |

For servers not in the catalog: treat as custom, prompt for command + args.

---

## Workflow

### 1 · Identify Server

**If server is in the well-known catalog:**
- Resolve command, args template, required env vars, and tool list from catalog above
- Note any args that need user input (e.g. `filesystem` needs allowed directory path)

**If `--path` is provided (custom server from `/mcp:build`):**
- Read `<path>/README.md` to extract: tool list, env vars, run command
- Detect language: `package.json` present → TypeScript (`node dist/index.js`), `server.py` present → Python
- Confirm the server has been built: check `dist/index.js` exists (TS) or `server.py` exists (Py)
- If not built: run `npm install && npm run build` or `pip install -e .` first

**If server is unknown and no `--path`:**
Use AskUserQuestion to collect:
> What command runs this server? (e.g. `npx my-server`, `python /path/server.py`)

---

### 2 · Collect Environment Variables

Read current env:
```bash
# Check if vars already set in shell or .env
printenv | grep -E "<VAR_NAMES>" 2>/dev/null
```

For each required env var not already provided via `--env` or found in environment:

Use AskUserQuestion:
> `<VAR_NAME>` is required for <server>. Enter value (will be stored in Claude settings):

For sensitive values (keys, tokens, passwords): note they will be stored in the settings file — remind the user to keep that file secure.

---

### 3 · Read Current Settings

Resolve settings file path:
```bash
# user scope
SETTINGS=~/.claude/settings.json

# project scope
SETTINGS=.claude/settings.json
```

```bash
# Read existing settings (create if missing)
cat "$SETTINGS" 2>/dev/null || echo "{}"
```

Parse the `mcpServers` block. If the server name already exists:

Use AskUserQuestion:
> `<name>` already exists in settings. → Overwrite / Keep existing / Abort

---

### 4 · Build Config Entry

Construct the settings entry:

**Well-known TypeScript/npx servers:**
```json
{
  "<name>": {
    "command": "npx",
    "args": ["-y", "<package>", "<any-path-args>"],
    "env": {
      "<VAR>": "<value>"
    }
  }
}
```

**Custom TypeScript server:**
```json
{
  "<name>": {
    "command": "node",
    "args": ["<absolute-path>/dist/index.js"],
    "env": { ... }
  }
}
```

**Custom Python server:**
```json
{
  "<name>": {
    "command": "python",
    "args": ["<absolute-path>/server.py"],
    "env": { ... }
  }
}
```

Use absolute paths for all local server references.

---

### 5 · Write Settings

Merge the new entry into the existing settings file — do not overwrite other keys:

```bash
# Use python for reliable JSON merge (always available)
python3 -c "
import json, sys

with open('$SETTINGS', 'r') as f:
    settings = json.load(f) if f.read(1) else {}

settings.setdefault('mcpServers', {})
settings['mcpServers']['<name>'] = <entry>

with open('$SETTINGS', 'w') as f:
    json.dump(settings, f, indent=2)
print('Written.')
" 2>&1
```

Show the written entry to the user for confirmation.

---

### 6 · Validate Connection

Attempt to launch the server and list its tools:

```bash
# Quick smoke test — launch server, send initialize + tools/list, kill
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0"}}}
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
| timeout 10 <command> <args> 2>/dev/null | head -50
```

**If validation succeeds:** parse and display the actual tool list from the server response.

**If validation fails:**
- Check the error: missing dependency, wrong path, bad env var, port conflict
- Provide a specific fix suggestion
- Ask: retry after fix, or skip validation and save anyway

---

### 7 · Document Available Tools

Print a tool reference card:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ MCP server configured: <name>
  Scope: <user|project>
  Settings: <settings file path>

  🔧 Available Tools
  ┌─────────────────────────────────────┐
  │ tool_name       Brief description   │
  │ tool_name_2     Brief description   │
  └─────────────────────────────────────┘

  💡 Usage in Claude
  These tools are now available in your next Claude Code session.
  Reference them in prompts or they'll be auto-used when relevant.

  Example prompts:
  <2-3 example prompts showing how to use the tools>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## --remove mode

If `--remove` flag passed:

```bash
python3 -c "
import json
with open('$SETTINGS') as f: s = json.load(f)
removed = s.get('mcpServers', {}).pop('<name>', None)
with open('$SETTINGS', 'w') as f: json.dump(s, f, indent=2)
print('Removed.' if removed else 'Not found.')
"
```

Confirm removal and remind that the next Claude session will no longer have access to those tools.

---

## Quick Examples

```bash
# Well-known servers
/mcp:setup playwright
/mcp:setup postgres --env DATABASE_URL=postgresql://localhost/mydb
/mcp:setup github --env GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxx
/mcp:setup filesystem
/mcp:setup brave-search

# Project-scoped (checked into repo, shared with team)
/mcp:setup playwright --scope project
/mcp:setup postgres --scope project --env DATABASE_URL=postgresql://localhost/mydb

# Custom server built with /mcp:build
/mcp:setup my-api-server --path ./mcp-servers/my-api-server --name my-api

# Remove
/mcp:setup playwright --remove
```
