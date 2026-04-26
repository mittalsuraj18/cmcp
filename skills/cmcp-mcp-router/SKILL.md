---
name: cmcp-mcp-router
description: Route all MCP work through the code-mode-mcp proxy. Discover with search first, then invoke with execute; do not call upstream MCP servers directly.
---

# Route MCP through cmcp

Use this skill in any AI client that has `cmcp` installed as an MCP server. The goal is not to teach the project; the goal is to make the client use the `code-mode-mcp` proxy correctly.

## When to use

Use this skill whenever a task requires MCP tools and the client has access to the `code-mode-mcp` server.

`cmcp` exposes a single gateway server with two tools:
- `search`: inspect the available upstream MCP tools and schemas by running TypeScript against the tool catalog.
- `execute`: call upstream MCP tools by running typed async TypeScript.

## Non-negotiable rules

1. Always run `search` before calling an MCP tool through `execute`.
2. Always call upstream MCP tools through `execute` when `code-mode-mcp` is available.
3. Do not call provider-specific MCP servers directly unless `code-mode-mcp` is unavailable or the server is explicitly listed as an exception.
4. Do not guess tool names, schema fields, or server globals. Inspect metadata first with `search`.
5. Return only the fields needed for the task to avoid context bloat.
6. Prefer one `execute` call for a coherent tool workflow; use `Promise.all` only for independent calls.

## Client tool naming

Different clients display the same cmcp tools with different names. Treat these as equivalent:

| Client | Discovery tool | Invocation tool |
| --- | --- | --- |
| Oh My Pi | `mcp_code_mode_mcp_search` | `mcp_code_mode_mcp_execute` |
| Claude Code / Claude Desktop | `code-mode-mcp.search` or `search` under `code-mode-mcp` | `code-mode-mcp.execute` or `execute` under `code-mode-mcp` |
| OpenCode | `search` under the configured `code-mode-mcp` MCP server | `execute` under the configured `code-mode-mcp` MCP server |
| Other MCP clients | the cmcp server's `search` tool | the cmcp server's `execute` tool |

If both cmcp and a direct upstream provider tool are visible, use cmcp unless the direct server is in the exceptions section.

## Standard workflow

### 1. Discover with `search`

Find candidate tools:

```typescript
return tools
  .filter(t =>
    t.name.includes("issue") ||
    t.description.toLowerCase().includes("issue") ||
    t.server.toLowerCase().includes("github")
  )
  .map(t => ({
    server: t.server,
    name: t.name,
    description: t.description,
  }));
```

Inspect a schema before invoking:

```typescript
const tool = tools.find(t => t.server === "github" && t.name === "create_issue");
return tool
  ? { server: tool.server, name: tool.name, input_schema: tool.input_schema }
  : null;
```

### 2. Invoke with `execute`

Single tool call:

```typescript
const issue = await github.create_issue({
  owner: "acme",
  repo: "platform",
  title: "Bug: login fails",
  body: "Steps to reproduce...",
});

return { number: issue.number, url: issue.html_url };
```

Chained calls in one workflow:

```typescript
const open = await github.list_issues({
  owner: "acme",
  repo: "platform",
  state: "open",
});

await slack.post_message({
  channel: "#eng-alerts",
  text: `Open issues: ${open.length}`,
});

return { open_count: open.length, notified: true };
```

Parallel independent calls:

```typescript
const [issues, deploys] = await Promise.all([
  github.list_issues({ owner: "acme", repo: "platform", state: "open" }),
  vercel.list_deployments({ project_id: "prj_123" }),
]);

return { issue_count: issues.length, deployment_count: deploys.length };
```

## Server global naming

Server names are sanitized into TypeScript globals inside `execute`:

- `my-server` becomes `my_server`
- `github-enterprise` becomes `github_enterprise`

Use `search` to confirm the server name, then use the sanitized identifier in `execute`.

## Missing tool or server

If `search` does not show the server or tool you need:

1. Check the cmcp server list outside the client:

   ```bash
   cmcp list --short
   ```

2. Add the missing upstream server to cmcp:

   ```bash
   # HTTP MCP server
   cmcp add canva https://mcp.canva.com/mcp

   # stdio MCP server
   cmcp add --transport stdio github -- npx -y @modelcontextprotocol/server-github
   ```

3. If the client does not show the cmcp proxy itself, install or reinstall cmcp into the client:

   ```bash
   cmcp install
   ```

4. Retry `search`. cmcp hot-reloads config changes, so newly added servers should appear on later calls without restarting most clients.

## Installing these instructions into clients

Use the most native instruction mechanism each client supports:

- Oh My Pi: install this file at `~/.omp/agent/skills/cmcp-mcp-router/SKILL.md` or add this project skill directory to the OMP skill source.
- Claude Code: copy this directory to `~/.claude/skills/cmcp-mcp-router/` for a personal skill, or `.claude/skills/cmcp-mcp-router/` for a project skill.
- OpenCode: paste the Client Instruction Block below into `AGENTS.md`, or reference a prompt file containing it from an OpenCode agent in `opencode.json`.
- Other clients: add the Client Instruction Block to the client's system, developer, project, or agent instructions.

### Client Instruction Block

```text
When MCP tools are needed and the `code-mode-mcp` server is available, route all MCP work through cmcp. First use the cmcp `search` tool to discover the relevant upstream tool and inspect its schema. Then use the cmcp `execute` tool to call the upstream tool with typed TypeScript. Do not call provider-specific MCP servers directly unless cmcp is unavailable or the server cannot be proxied. Do not guess tool names, parameter names, or schemas; inspect them first. Return only the fields needed for the task.
```

## Exceptions

Some servers may need direct client registration instead of proxying through cmcp:

- Hook- or lifecycle-dependent servers that require tight integration with the host client.
- Interactive OAuth/browser callback flows that the proxy cannot complete.
- Client-local tools that are not MCP servers.

When an exception is required, state the reason and keep the boundary explicit: use cmcp for all proxy-compatible MCP servers, and call the direct tool only for the exception.

## Quick checklist

- Discover with `search`.
- Confirm the schema.
- Invoke with `execute`.
- Return minimal data.
- If missing, run `cmcp list --short`, add the server with `cmcp add`, then retry `search`.
