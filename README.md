# poor-mans-mcp

Convert any MCP server into a Claude Code plugin using [mcpc](https://github.com/apify/mcpc) CLI — for environments that don't support MCP directly, or when you want shell-scriptable MCP access.

## What this does

Instead of Claude calling MCP tools directly, this plugin teaches Claude to call them via `mcpc @session tools-call <name>` through the Bash tool. The MCP protocol is the same underneath — mcpc handles JSON-RPC, sessions, and auth — but it works anywhere you have a terminal.

The skill walks through a 5-phase process:

1. **Probe** — Find the server, figure out auth, get connected
2. **Discover** — Map out all tools, resources, and prompts
3. **Test** — Call tools to verify everything works
4. **Create Plugin** — Package capabilities into a Claude Code plugin
5. **Port Existing Skills** — Adapt an existing MCP plugin's skills for mcpc

## Install

### Prerequisites

```bash
npm install -g @apify/mcpc
```

### Via skills.sh (works with 50+ agents)

```bash
npx skills add aniruddha-adhikary/poor-mans-mcp
```

Works with Claude Code, Cursor, Cline, OpenCode, and more.

### As a Claude Code plugin (from marketplace)

```
/plugin marketplace add https://github.com/aniruddha-adhikary/poor-mans-mcp.git
/plugin install poor-mans-mcp
```

### As a Claude Code plugin (local)

```bash
claude --plugin-dir /path/to/poor-mans-mcp
```

### As a standalone skill

Copy `skills/convert-mcp/SKILL.md` into your project's `.claude/skills/convert-mcp/` directory.

### For other agents (Codex, Gemini CLI, etc.)

Copy `.agents/skills/convert-mcp.md` into your project's `.agents/skills/` directory.

## Usage

Just ask Claude:

- "Connect to the Figma MCP server via mcpc"
- "Convert this MCP to a plugin"
- "Probe https://mcp.example.com/mcp and create skills for it"
- "Port the official Figma plugin to use mcpc"

The skill triggers automatically when you mention MCP servers, mcpc, or plugin conversion.

## Real-world examples

### Context7 (anonymous, no auth)

```bash
mcpc connect https://mcp.context7.com/mcp @context7 --no-profile
mcpc @context7 tools-list
mcpc @context7 tools-call resolve-library-id query:="react hooks" libraryName:="react"
```

### Figma (OAuth with allowlisted client name)

```bash
# Register (name must be exactly "Claude Code (figma)")
curl -X POST https://api.figma.com/v1/oauth/mcp/register \
  -H "Content-Type: application/json" \
  -d '{"client_name":"Claude Code (figma)","redirect_uris":["http://127.0.0.1:8000/callback"],"grant_types":["authorization_code","refresh_token"],"response_types":["code"],"token_endpoint_auth_method":"none"}'

# Login with returned credentials
mcpc login https://mcp.figma.com/mcp --client-id <ID> --client-secret <SECRET>

# Connect and use
mcpc connect https://mcp.figma.com/mcp @figma
mcpc @figma tools-list
mcpc @figma tools-call whoami
```

### Local stdio server

```bash
mcpc connect .vscode/mcp.json:filesystem @fs
mcpc @fs tools-list
```

## Plugin structure

```
poor-mans-mcp/
├── .claude-plugin/
│   └── plugin.json              # Claude Code plugin manifest
├── skills/
│   └── convert-mcp/
│       ├── SKILL.md             # Main skill (5-phase workflow)
│       └── references/
│           └── mcpc-advanced.md # Proxy, async tasks, scripting, debugging
├── .agents/
│   └── skills/
│       └── convert-mcp.md      # Universal agents format
├── LICENSE
└── README.md
```

## Auth patterns supported

| Server Type | Method | Example |
|-------------|--------|---------|
| Anonymous | `--no-profile` | Context7 |
| Optional API key | `--header "Authorization: Bearer <key>"` | Context7 with key |
| OAuth 2.1 | `mcpc login <url>` | Standard servers |
| Allowlisted DCR | Manual curl to registration endpoint | Figma |
| Pre-registered | `--client-id` + `--client-secret` | Developer portal apps |
| Bearer token | `--header "Authorization: Bearer <PAT>"` | Personal access tokens |
| Local stdio | Config file entry | Filesystem, database servers |

## Gotchas

Some MCP servers have undocumented restrictions on OAuth Dynamic Client Registration (DCR):

- **Figma** allowlists the client name `"Claude Code (figma)"` — any other name returns 403. See [this discussion](https://github.com/anomalyco/opencode/issues/988#issuecomment-4022520800) for the workaround.
- Other servers may have similar restrictions. If `mcpc login` fails with 403, try manual DCR with different client names or check the server's GitHub issues.

## License

MIT — see [LICENSE](LICENSE).
