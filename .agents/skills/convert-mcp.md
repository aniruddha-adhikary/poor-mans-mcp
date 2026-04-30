---
name: convert-mcp
description: "Convert any MCP server into a Claude Code plugin that uses mcpc CLI for tool calls. Use this skill whenever the user wants to connect to an MCP server via mcpc, probe an MCP server's capabilities, create a plugin from an MCP server, port an existing MCP plugin to use mcpc, or says things like 'connect to this MCP', 'make this MCP work without direct MCP support', 'convert this MCP to a plugin', 'probe this MCP server', or provides an MCP server URL. Also use when the user mentions mcpc setup, MCP authentication issues, or wants CLI access to MCP tools. Even if the user just says 'I want to use X MCP server' without mentioning mcpc or plugins, this skill likely applies."
---

# Convert MCP Server to mcpc Plugin

Turn any MCP server into a Claude Code plugin that works through the `mcpc` CLI — useful when your environment doesn't support direct MCP connections, when you want shell-scriptable MCP access, or when you want to package MCP capabilities as reusable skills.

The approach: instead of Claude calling MCP tools directly, the plugin's skills instruct Claude to call them via `mcpc @session tools-call <name>` through the Bash tool. The MCP protocol is the same underneath — mcpc handles the JSON-RPC, sessions, and auth — but the skill layer makes it work anywhere you have a terminal.

## Prerequisites

- `mcpc` installed globally: `npm install -g @apify/mcpc`

## Overview

5 phases, each building on the last:

1. **Probe** — Find the server, figure out auth, get connected
2. **Discover** — Map out all tools, resources, and prompts
3. **Test** — Call tools to verify everything works (and learn quirks)
4. **Create Plugin** — Package what you learned into a Claude Code plugin
5. **Port Existing Skills** (optional) — If an official plugin exists, adapt its skills for mcpc

---

## Phase 1: Probe & Authenticate

This is often the trickiest part. MCP servers use various auth mechanisms and some have undocumented restrictions. The strategy: start simple, escalate if needed.

### 1.1 Find the server URL

Look for it in:
- `.mcp.json` or `.vscode/mcp.json` in the project (most common)
- A `server.json` manifest with `remotes[].url`
- The user providing it directly
- The server's documentation or GitHub repo

**Remote HTTP servers** have URLs like `https://mcp.example.com/mcp`.

**Local stdio servers** are defined in config files and launched as subprocesses:
```json
{
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["-y", "@example/mcp-server"],
      "env": { "API_KEY": "${MY_API_KEY}" }
    }
  }
}
```

For stdio servers, connect using the config file syntax:
```bash
mcpc connect .mcp.json:my-server @my-server

# Or connect ALL servers in a config file at once:
mcpc connect .mcp.json
```

### 1.2 Try anonymous connection first

Always start here — many servers don't require auth:

```bash
mcpc connect <server-url> @<session-name> --no-profile
```

If this succeeds, skip to Phase 2. If you get 401/403/connection errors, continue below.

### 1.3 Probe OAuth metadata

When anonymous connection fails, the server likely uses OAuth 2.1. Probe its infrastructure to understand what it expects:

```bash
# Send an unauthenticated request to see the 401 challenge
# The WWW-Authenticate header reveals the auth structure
curl -s -D- -X POST <server-url> \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"mcpc","version":"1.0"}}}'

# Fetch Protected Resource Metadata (RFC 9728)
# This tells you which authorization server to use
curl -s <server-origin>/.well-known/oauth-protected-resource | python3 -m json.tool

# Fetch Authorization Server Metadata
# Extract the authorization_servers URL from the resource metadata above
curl -s <auth-server-url>/.well-known/oauth-authorization-server | python3 -m json.tool
```

The AS metadata reveals the server's auth requirements:
- `registration_endpoint` — whether Dynamic Client Registration (DCR) is supported
- `token_endpoint_auth_methods_supported` — `["none"]` means public clients OK; `["client_secret_post"]` means you need a secret
- `client_id_metadata_document_supported` — whether CIMD (URL-based client IDs) works
- `code_challenge_methods_supported` — should include `S256` for PKCE
- `scopes_supported` — what permissions to request

### 1.4 Authenticate

Try these in order — each one handles a different server personality:

**Option A: Standard mcpc login**
```bash
mcpc login <server-url>
# Optionally request specific scopes:
mcpc login <server-url> --scope "read write"
```
Uses mcpc's built-in CIMD → DCR fallback. Opens a browser for OAuth consent. Works for most well-behaved servers.

**Option B: Pre-registered client credentials**
If the server has a developer portal where you can register an OAuth app:
```bash
mcpc login <server-url> --client-id <ID> --client-secret <SECRET>
```

**Option C: Manual DCR**
When `mcpc login` fails (often with 403), the server's DCR endpoint may have restrictions. Try registering directly:
```bash
curl -X POST <registration-endpoint> \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "<try-different-names>",
    "redirect_uris": ["http://127.0.0.1:8000/callback"],
    "grant_types": ["authorization_code", "refresh_token"],
    "response_types": ["code"],
    "token_endpoint_auth_method": "none"
  }'
```

**Why this matters:** Some servers allowlist specific client names. For example, Figma's DCR only accepts `"Claude Code (figma)"` — everything else gets 403. If you hit this, try `"Claude Code (<server-name>)"` as a pattern, or search GitHub issues for the server.

After DCR succeeds, use the returned `client_id` and `client_secret` with Option B.

**Option D: Bearer token**
If OAuth isn't viable, use a Personal Access Token or API key:
```bash
mcpc connect <server-url> @<session-name> \
  --header "Authorization: Bearer <TOKEN>" --no-profile
```

### 1.5 Connect and verify

```bash
mcpc connect <server-url> @<session-name>

# Verify — try whoami if available, otherwise just list tools
mcpc @<session-name> tools-call whoami 2>/dev/null || mcpc @<session-name> tools-list
```

### 1.6 Document what worked

Record the auth flow so it can be reproduced in the setup skill later:
- Which auth option succeeded (A/B/C/D)
- Any client name restrictions or registration quirks
- Required scopes
- Whether the server needs a specific `token_endpoint_auth_method`

---

## Phase 2: Discover Capabilities

Now map everything the server offers. This becomes the foundation for your plugin's skills.

```bash
# Server info, capabilities, and instructions
mcpc @<session-name>

# All tools with full schemas (names, args, descriptions)
mcpc @<session-name> tools-list --full

# Get a single tool's full schema
mcpc @<session-name> tools-get <tool-name>

# Resources (documentation, data the server exposes)
mcpc @<session-name> resources-list

# Prompts (reusable prompt templates)
mcpc @<session-name> prompts-list

# JSON output for scripting/parsing (works on any command)
mcpc --json @<session-name> tools-list
```

If the server has many tools, use `mcpc grep` to search across them:
```bash
mcpc @<session-name> grep "search"
```

**Read the server instructions carefully** — they often contain critical usage guidance, workflow recommendations, and context about how tools relate to each other.

**Read key resources** — many servers expose documentation as resources. These are gold for writing skill reference docs:
```bash
mcpc @<session-name> resources-read "<resource-uri>"
```

**Categorize the tools** as you discover them:
- **Read-only** (safe to call freely): queries, lookups, metadata
- **Write/mutating**: creates, updates, deletes
- **Destructive** (use with care): bulk deletes, overwrites
- **No-args** (good for testing): health checks, whoami, list operations

---

## Phase 3: Test Tools

Test systematically to verify the connection works end-to-end and to learn the server's quirks before building skills around it.

### Testing strategy

1. **Start with no-arg and read-only tools** — these are safe and reveal whether auth works
2. **Try each argument pattern** — simple strings, numbers, booleans, complex JSON via stdin
3. **Use `--timeout` for slow tools** — default is 300s, increase with `mcpc --timeout 600 @<name> tools-call ...`
4. **Watch for rate limits** — some servers have plan-based quotas. If you hit limits during testing, you'll know to document them
4. **Note error patterns** — what do errors look like? Are they JSON? Do they include helpful messages?
5. **Test write tools last** — and only if the user is OK with creating test data

```bash
# No-arg tools first
mcpc @<session-name> tools-call <tool-name>

# Simple arguments
mcpc @<session-name> tools-call <tool-name> arg1:="value1" arg2:="value2"

# Complex arguments via stdin (for special characters, long strings, nested JSON)
echo '{"arg1":"value","arg2":"complex value"}' | mcpc @<session-name> tools-call <tool-name>

# Resources
mcpc @<session-name> resources-read "<resource-uri>"

# Prompts
mcpc @<session-name> prompts-get <prompt-name>
```

### What to record

For each tool, note:
- Does it work? What does success look like?
- What does failure look like?
- Any surprising behavior or undocumented requirements?
- Rate limit errors? Plan/tier restrictions?
- How long does it take? (some tools are slow)

---

## Phase 4: Create Plugin

### 4.1 Choose where to create skills

**Default: project-local skills** — these load automatically in the current project:
```
<project>/
├── .claude/
│   └── skills/
│       ├── <server>-setup/
│       │   └── SKILL.md
│       └── <capability-group>/
│           ├── SKILL.md
│           └── references/
│               └── patterns.md
└── .agents/
    └── skills/
        ├── <server>-setup.md      # Universal agents format
        └── <capability-group>.md
```

Skills in `.claude/skills/` are automatically available in the project — no `--plugin-dir` or `/plugin install` needed. Use this when the MCP integration is for this specific project.

**Alternative: standalone plugin** — for sharing across projects or via marketplace:
```
my-mcp-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── ...
└── .agents/
    └── skills/
        └── ...
```

Standalone plugins require `claude --plugin-dir ./my-mcp-plugin` or marketplace installation. Use this when packaging for distribution.

### 4.2 Create the setup skill

Every mcpc integration needs a setup skill. This is the first thing users will need. Include:
- Prerequisites (`mcpc` installation)
- Step-by-step auth flow (from Phase 1 — what actually worked)
- Connection command with the exact session name
- Verification command
- Troubleshooting for common failures

For standalone plugins, also create a manifest at `.claude-plugin/plugin.json`:
```json
{
  "name": "<plugin-name>",
  "description": "<what this plugin does> via mcpc CLI",
  "version": "1.0.0",
  "author": { "name": "<author>" }
}
```

### 4.3 Create capability skills

Group related tools into skills based on user intent, not 1:1 with tools. A "design reading" skill might cover `get_screenshot`, `get_metadata`, and `get_design_context` together because users reach for them as a group.

Each SKILL.md needs:

**Frontmatter** with a description that triggers on user intent:
```yaml
---
description: "<what this does>. Use when <user intent descriptions>."
---
```

**mcpc invocation block** immediately after frontmatter — this is the key adaptation:
````markdown
## mcpc Tool Invocation

All tool calls use `mcpc` CLI via the Bash tool:
```bash
mcpc @<session-name> tools-call <tool_name> arg1:="value1" arg2:="value2"
# For complex arguments, pipe via stdin:
echo '{"key":"value"}' | mcpc @<session-name> tools-call <tool_name>
```
````

**Tool documentation** for each tool in the group:
- What it does and when to use it
- Required vs optional arguments with types
- Example invocations (copy from Phase 3 testing)
- Gotchas discovered during testing
- How to interpret results

**Reference docs** for detailed patterns — keep SKILL.md under ~500 lines and put detailed reference material in `references/` files.

### 4.4 Copy to universal format

For each SKILL.md, also create a copy at `.agents/skills/<skill-name>.md` so the plugin works with Codex, Gemini CLI, and other agents that read `.agents/skills/`.

### 4.6 Test the plugin

```bash
claude --plugin-dir ./my-mcp-plugin
```

Try each skill and verify tools get called correctly through mcpc.

---

## Phase 5: Port Existing Skills (Optional)

If there's already a Claude Code plugin for this MCP server (check the plugin marketplace or GitHub), you can port its skills instead of writing from scratch. This preserves battle-tested instructions and just changes the invocation layer.

### 5.1 Copy and clean

```bash
cp -r <existing-plugin-dir> <new-plugin-dir>
rm -rf <new-plugin-dir>/.git <new-plugin-dir>/.github
rm -f <new-plugin-dir>/.mcp.json      # Remove direct MCP config
rm -f <new-plugin-dir>/figma-power/mcp.json  # Remove any nested MCP configs
```

### 5.2 Update the manifest

Edit `.claude-plugin/plugin.json` — new name, description mentioning mcpc.

### 5.3 Add mcpc invocation blocks

For every SKILL.md, insert the mcpc invocation block after the frontmatter. Prefix `[mcpc]` to each skill's description so users know it's the mcpc variant.

When there are many skills, use parallel subagents — one per skill or group of skills — to do this efficiently.

### 5.4 Replace MCP references

Search and replace across all `.md` files. Common patterns:
- `"<Server> MCP server must be connected"` → `"mcpc must be connected to <Server> (\`mcpc @<session>\`)"`
- `"MCP tool"` / `"MCP tools"` → `"mcpc tool"` / `"mcpc tools"`
- `"Call the MCP tool"` → `"Call via mcpc"`
- `"<Server> MCP server"` → `"<Server> (via mcpc)"`
- `"MCP client"` → `"mcpc"`
- ToolSearch / deferred tool loading sections → replace with "All tools are called via `mcpc @<session> tools-call`"

Use parallel subagents for large plugins with many files.

### 5.5 Add setup skill

Create a setup skill with the auth flow from Phase 1 if one doesn't exist.

### 5.6 Add universal format

Copy skills to `.agents/skills/` for cross-agent compatibility.

### 5.7 Verify no stale references

```bash
grep -rn "MCP" skills/ --include="*.md" | grep -iv "mcpc"
```

Review any remaining hits — some are legitimate (references to other MCP servers, product names like `figma-desktop`).

---

## Reference: mcpc Argument Syntax

| Pattern | Example | Notes |
|---------|---------|-------|
| Simple string | `name:="hello"` | Auto-parsed as string |
| Number | `count:=10` | Auto-parsed as number |
| Boolean | `enabled:=true` | Auto-parsed as boolean |
| Force string | `id:='"123"'` | JSON string literal |
| Inline JSON | `'{"key":"value"}'` | First arg starts with `{` |
| Stdin | `echo '{}' \| mcpc ...` | For special characters |
| Shell vars | `"query:=${VAR}"` | Double-quote the whole arg |

## Reference: Common Auth Patterns

| Server Type | Auth Method | Notes |
|-------------|------------|-------|
| Anonymous | `--no-profile` | No auth needed |
| Anonymous + optional key | `--header "Authorization: Bearer <key>"` | Works without, key unlocks higher limits |
| OAuth 2.1 (standard) | `mcpc login <url>` | CIMD/DCR flow, opens browser |
| OAuth 2.1 (pre-registered) | `--client-id <id> --client-secret <secret>` | From developer portal |
| OAuth 2.1 (allowlisted DCR) | Manual curl to registration endpoint | Some servers restrict client names |
| Token-based | `--header "Authorization: Bearer <token>"` | PAT or API key |
| Local stdio | Config file entry | `mcpc connect .mcp.json:<name> @<name>` |

### Real-World Examples

**Context7** (anonymous, optional API key):
```bash
# Without key
mcpc connect https://mcp.context7.com/mcp @context7 --no-profile

# With API key for higher limits
mcpc connect https://mcp.context7.com/mcp @context7 \
  --header "Authorization: Bearer ctx7sk-..." --no-profile
```

**Figma** (OAuth with allowlisted client name):
```bash
# Step 1: Register (name MUST be exactly "Claude Code (figma)")
curl -X POST https://api.figma.com/v1/oauth/mcp/register \
  -H "Content-Type: application/json" \
  -d '{"client_name":"Claude Code (figma)","redirect_uris":["http://127.0.0.1:8000/callback"],"grant_types":["authorization_code","refresh_token"],"response_types":["code"],"token_endpoint_auth_method":"none"}'

# Step 2: Login with returned credentials
mcpc login https://mcp.figma.com/mcp --client-id <ID> --client-secret <SECRET>

# Step 3: Connect
mcpc connect https://mcp.figma.com/mcp @figma
```

**Local stdio server** (from config file):
```bash
mcpc connect .vscode/mcp.json:filesystem @fs
```

## Reference: Session Management

- **List all sessions**: `mcpc` (no args)
- **Restart a crashed session**: `mcpc @<name> restart`
- **Close a session**: `mcpc @<name> close`
- **Check logs**: `~/.mcpc/logs/bridge-@<name>.log`
- **Clean stale sessions**: `mcpc clean sessions`
- **Search tools across sessions**: `mcpc grep "pattern"`
- **Interactive mode**: `mcpc @<name> shell`

## Reference: Advanced mcpc Features

For proxy mode, async tasks, resource subscriptions, JSON scripting pipelines, named OAuth profiles, debugging, and profile management, see [references/mcpc-advanced.md](./references/mcpc-advanced.md).

## Reference: Troubleshooting

- **403 on DCR**: Server may allowlist client names. Search GitHub issues for workarounds.
- **403 on login**: Server may not support CIMD or public clients. Try manual DCR or `--client-id`.
- **Session disconnected**: `mcpc @<name> restart` — sessions auto-recover when the server comes back.
- **Rate limited**: Check the server's plan/tier. Some servers return upgrade links in error messages.
- **Stdio server won't start**: Check the command exists, env vars are set, and stderr logs in `~/.mcpc/logs/`.
- **"Forbidden" (not JSON)**: The server returned a raw text error, not a JSON-RPC error. This usually means the endpoint or auth is wrong.
