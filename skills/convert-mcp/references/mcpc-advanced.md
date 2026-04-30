# mcpc Advanced Features

> **Note:** Read this reference when you need features beyond the basics covered in the main SKILL.md — proxying sessions, async tasks, resource subscriptions, debugging, or profile management.

## Table of Contents

- [Proxy Mode](#proxy-mode)
- [Async Tasks](#async-tasks)
- [Resource Subscriptions & Templates](#resource-subscriptions--templates)
- [JSON Mode for Scripting](#json-mode-for-scripting)
- [OAuth Advanced Options](#oauth-advanced-options)
- [Debugging](#debugging)
- [Profile Management](#profile-management)

---

## Proxy Mode

Expose an mcpc session as a local MCP server — useful for giving other tools authenticated access without sharing credentials:

```bash
# Start a proxy on localhost:8080
mcpc connect https://mcp.example.com @relay --proxy 8080

# Other tools connect to the proxy like a regular MCP server
mcpc connect localhost:8080 @sandboxed

# Protect the proxy with a bearer token
mcpc connect https://mcp.example.com @relay --proxy 8081 --proxy-bearer-token secret123
```

The proxy never exposes your OAuth tokens to downstream clients. Useful for AI sandboxing — the agent connects to the proxy and can only use pre-authorized credentials.

## Async Tasks

Some MCP servers run tools as background tasks. mcpc supports the full task lifecycle:

```bash
# Run a tool as a task (waits for completion, shows progress)
mcpc @<session> tools-call long-running-job input:="data" --task

# Start a task and return immediately
mcpc @<session> tools-call long-running-job input:="data" --detach

# List active tasks
mcpc @<session> tasks-list

# Check status
mcpc @<session> tasks-get <taskId>

# Get result (blocks until complete)
mcpc @<session> tasks-result <taskId>

# Cancel
mcpc @<session> tasks-cancel <taskId>
```

Press **ESC** during `--task` to detach and get the task ID for later retrieval.

## Resource Subscriptions & Templates

Beyond basic `resources-list` and `resources-read`:

```bash
# List resource URI templates (parameterized resources)
mcpc @<session> resources-templates-list

# Subscribe to resource changes (in shell mode, get real-time notifications)
mcpc @<session> resources-subscribe "https://api.example.com/data"

# Unsubscribe
mcpc @<session> resources-unsubscribe "https://api.example.com/data"
```

## JSON Mode for Scripting

Add `--json` to any command for machine-readable output. JSON follows the MCP specification strictly.

```bash
# Get tools as JSON, pipe to jq
mcpc --json @<session> tools-list | jq '.[].name'

# Chain tools across sessions
mcpc --json @apify tools-call search keywords:="scraper" \
  | jq '.content[0].text | fromjson | .items[0].id' \
  | xargs -I {} mcpc @apify tools-call get-actor actorId:="{}"

# Save all tool schemas
for tool in $(mcpc --json @server tools-list | jq -r '.[].name'); do
  mcpc --json @server tools-get "$tool" > "schemas/$tool.json"
done
```

## OAuth Advanced Options

### Custom CIMD URL

Override mcpc's default Client ID Metadata Document:
```bash
mcpc login https://mcp.example.com --client-metadata-url https://my-app.com/client.json
```

### Force DCR (skip CIMD)

```bash
mcpc login https://mcp.example.com --no-client-metadata-url
```

### Named profiles

Use different accounts with the same server:
```bash
mcpc login https://mcp.example.com --profile work
mcpc login https://mcp.example.com --profile personal

mcpc connect https://mcp.example.com @work-session --profile work
mcpc connect https://mcp.example.com @personal-session --profile personal
```

### Request specific scopes

```bash
mcpc login https://mcp.example.com --scope "files:read files:write"
```

## Debugging

### Verbose mode

See full JSON-RPC messages and HTTP details:
```bash
mcpc --verbose @<session> tools-call <tool>
```

### Ping

Check if a session's server is alive:
```bash
mcpc @<session> ping
```

### Server logging level

Adjust server-side log verbosity:
```bash
mcpc @<session> logging-set-level debug
mcpc @<session> logging-set-level error
```

### Self-signed certificates

For dev servers with self-signed TLS certs:
```bash
mcpc connect https://dev.local:8443 @dev --insecure
```

### Bridge logs

Each session's background process logs to:
```
~/.mcpc/logs/bridge-@<session>.log
```

## Profile Management

```bash
# List all sessions and profiles
mcpc

# Remove a saved OAuth profile
mcpc logout https://mcp.example.com
mcpc logout https://mcp.example.com --profile work

# Clean everything
mcpc clean all

# Clean just sessions (kill bridges, delete records)
mcpc clean sessions

# Clean just profiles
mcpc clean profiles

# Clean just logs
mcpc clean logs
```
