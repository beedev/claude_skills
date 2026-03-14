---
allowed-tools: [Bash, Read]
description: "Search and fetch LLM-optimized API reference docs for external providers via chub CLI"
---

# /chub — External API Reference Lookup

## Purpose
Invoke the chub CLI to search and retrieve up-to-date, LLM-optimized documentation for external APIs and packages. Use this before implementing any external service integration to get authoritative request/response shapes, auth headers, and error codes.

## Usage

Parse `$ARGUMENTS` to determine which subcommand to run.

### If argument starts with "search"

Search the chub registry for available docs.

**Examples:**
- `/chub search stripe` — find Stripe-related docs
- `/chub search openai` — find OpenAI docs
- `/chub search --tags auth` — filter by tag
- `/chub search` — list all available docs

**Action:**
1. Run: `chub search <query> [--tags <tags>] [--lang <language>] [--limit <n>]`
2. Display results in a clean table: ID, type, languages, description
3. Suggest relevant `get` commands for the top matches

### If argument starts with "get"

Fetch specific API docs by ID.

**Examples:**
- `/chub get stripe/payments --lang py` — Stripe Payments for Python
- `/chub get openai/chat --lang js` — OpenAI Chat API for JavaScript
- `/chub get anthropic/messages` — Anthropic Messages API (default lang)
- `/chub get stripe/payments openai/chat` — fetch multiple docs

**Action:**
1. Run: `chub get <ids...> [--lang <language>] [--version <version>]`
2. Display the returned documentation content
3. Summarize key patterns:
   - **Auth**: how to authenticate (API keys, OAuth, headers)
   - **Core endpoints/methods**: primary operations available
   - **Request shapes**: required and optional parameters
   - **Response shapes**: return types and structures
   - **Error codes**: common errors and handling
4. If `--full` flag is passed, fetch all files (not just entry point)

### If argument starts with "update"

Refresh the cached registry index.

**Action:**
1. Run: `chub update`
2. Confirm the cache was refreshed

### If argument starts with "annotate"

Attach agent notes to a doc or skill for future reference.

**Action:**
1. Run: `chub annotate <id> <note>`
2. Confirm the annotation was saved

### If argument starts with "feedback"

Rate a doc or skill.

**Action:**
1. Run: `chub feedback <id> <up|down> [comment]`
2. Confirm the rating was submitted

## Integration Notes

- **Before any external API call**: run `/chub get <provider>/<api> --lang <lang>` first — never guess at request shapes or auth headers
- **Unsure of the ID?**: run `/chub search <term>` first to find the exact doc ID
- **Multiple APIs in one task**: fetch all relevant docs upfront before planning implementation
- **Stale results?**: run `/chub update` to refresh the registry cache
