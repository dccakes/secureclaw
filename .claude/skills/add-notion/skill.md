---
name: add-notion
description: Add Notion as an MCP integration to NanoClaw. Use when user wants Hugo to read, search, or write to Notion pages and databases via WhatsApp. Triggers on "notion", "add notion", "notion integration", "notion mcp".
---

# Add Notion MCP Integration

Connects Hugo to Notion via the official Notion MCP server. Hugo can search pages, read databases, create and update content — all from WhatsApp.

**UX Note:** Use `AskUserQuestion` for all user-facing questions.

## Prerequisites

### 1. Check for existing token

```bash
grep NOTION_TOKEN .env 2>/dev/null && echo "Token found" || echo "No token"
```

If found, confirm with user: "You already have a NOTION_TOKEN configured. Want to keep it or reconfigure?" If keeping, skip to Step 2.

### 2. Get Notion token

**Use AskUserQuestion** to present this:

> You'll need a Notion integration token:
>
> 1. Go to https://www.notion.so/profile/integrations
> 2. Click **New integration**, name it (e.g., "Hugo")
> 3. Under **Capabilities**, enable Read, Update, and Insert content
> 4. Click **Submit** and copy the **Internal Integration Secret** (starts with `ntn_`)
>
> **Important:** Grant page access — open any Notion page you want Hugo to access → `...` menu → **Connections** → add your integration. Access to a top-level page grants access to all child pages.
>
> Add `NOTION_TOKEN=ntn_...` to the `.env` file in the project root, then let me know when done.

Wait for confirmation, then verify:

```bash
grep NOTION_TOKEN .env && echo "Token found"
```

### 3. Get page/database URLs

**Use AskUserQuestion** to ask:

> Which Notion pages or databases do you want Hugo to access? Share the URLs and I'll embed the IDs so Hugo knows where to look without searching every time.
>
> Common setups:
> - A "Tasks" database for task management
> - A "Notes" page for quick notes
> - A "Knowledge Base" for saved information

Collect URLs, extract IDs (the UUID after the last `-` in the URL, e.g. `https://notion.so/My-Page-abc123def456` → ID is `abc123def456`).

### 4. Test MCP server before touching code

```bash
echo '{"method": "tools/list"}' | NOTION_TOKEN=$(grep NOTION_TOKEN .env | cut -d= -f2) npx -y @notionhq/notion-mcp-server 2>/dev/null | head -20
```

If no output or error → token is invalid or npx failed. Check token and retry before proceeding.

## Implementation

### Step 1: Add NOTION_TOKEN to secret passthrough

Read `src/container-runner.ts` and find the `readSecrets()` function:

```typescript
function readSecrets(): Record<string, string> {
  return readEnvFile(['CLAUDE_CODE_OAUTH_TOKEN', 'ANTHROPIC_API_KEY']);
}
```

Add `NOTION_TOKEN`:

```typescript
function readSecrets(): Record<string, string> {
  return readEnvFile(['CLAUDE_CODE_OAUTH_TOKEN', 'ANTHROPIC_API_KEY', 'NOTION_TOKEN']);
}
```

### Step 2: Add Notion MCP server to agent runner

Read `container/agent-runner/src/index.ts` and find the `mcpServers` block. Add `notion` alongside `nanoclaw`:

```typescript
mcpServers: {
  nanoclaw: {
    command: 'node',
    args: [mcpServerPath],
    env: {
      NANOCLAW_CHAT_JID: containerInput.chatJid,
      NANOCLAW_GROUP_FOLDER: containerInput.groupFolder,
      NANOCLAW_IS_MAIN: containerInput.isMain ? '1' : '0',
    },
  },
  notion: {
    command: 'npx',
    args: ['-y', '--', '@notionhq/notion-mcp-server'],
    env: {
      NOTION_TOKEN: containerInput.secrets?.NOTION_TOKEN || '',
    },
  },
},
```

### Step 3: Allow Notion tools

In the same file, add `'mcp__notion__*'` to the `allowedTools` array:

```typescript
allowedTools: [
  'Bash',
  'Read', 'Write', 'Edit', 'Glob', 'Grep',
  'WebSearch', 'WebFetch',
  'Task', 'TaskOutput', 'TaskStop',
  'TeamCreate', 'TeamDelete', 'SendMessage',
  'TodoWrite', 'ToolSearch', 'Skill',
  'NotebookEdit',
  'mcp__nanoclaw__*',
  'mcp__notion__*'
],
```

### Step 4: Update group memory

Append to `groups/CLAUDE.md` (create if it doesn't exist). Include the specific database IDs collected in Prerequisites:

```markdown

## Notion

You have access to Notion via MCP tools.

**Reading:**
- `mcp__notion__search` - Search across all connected pages and databases
- `mcp__notion__retrieve-a-page` - Get a specific page by ID
- `mcp__notion__query-a-database` - Query a database with filters and sorts
- `mcp__notion__retrieve-block-children` - Get content blocks of a page

**Writing:**
- `mcp__notion__create-a-page` - Create a new page (in a database or as child of another page)
- `mcp__notion__update-page-properties` - Update page properties
- `mcp__notion__append-block-children` - Add content to a page

**Tips:**
- Use `mcp__notion__search` first to find pages when you don't know the ID
- Database entries are pages with properties — use `create-a-page` with `parent.database_id`
- Page content is made of blocks (paragraphs, headings, lists, etc.)

**Known pages/databases:**
<!-- Add IDs from user's URLs here, e.g.: -->
<!-- - Tasks: `abc123def456` — use for task tracking -->
<!-- - Notes: `def456abc123` — use for quick notes -->
```

Also append the same section to `groups/main/CLAUDE.md`.

Replace the `<!-- -->` comments with the actual IDs and labels from the user's URLs.

### Step 5: Build and restart

```bash
npm run build
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

Wait 3 seconds, then verify:

```bash
sleep 3 && tail -20 logs/nanoclaw.log | grep -iE "notion|mcp|error"
```

## Common Workflows

**Quick note capture**
> "Add to Notion notes: API rate limit is 100 req/min per user"

**Task management**
> "Add to my Notion tasks: Deploy v2.0 by Friday, high priority"

**Knowledge lookup**
> "Check Notion for our deployment checklist"

**Database query**
> "What tasks are due this week in Notion?"

## Troubleshooting

**401 / unauthorized errors**
- Verify token starts with `ntn_`: `grep NOTION_TOKEN .env`
- Rebuild the container and restart to pick up the new secret

**"object_not_found" errors**
- The integration doesn't have access to that page
- Open the page in Notion → `...` → **Connections** → add your integration
- Top-level page access covers all child pages

**MCP server not responding**

Test in isolation:
```bash
echo '{"method": "tools/list"}' | NOTION_TOKEN=$(grep NOTION_TOKEN .env | cut -d= -f2) npx -y @notionhq/notion-mcp-server 2>/dev/null | head -20
```

Check container logs:
```bash
cat groups/main/logs/container-*.log | tail -50
```

Verify npx works in the container:
```bash
docker run --rm nanoclaw-agent:latest npx --version
```

**Slow first response** — `npx -y` downloads the package on first run inside the container. To pre-install instead, add to `container/Dockerfile`:
```dockerfile
RUN npm install -g @notionhq/notion-mcp-server
```
Then change the agent runner `args` to `['notion-mcp-server']` with no `npx`.

## Removing Notion Integration

1. Remove `'mcp__notion__*'` from `allowedTools` in `container/agent-runner/src/index.ts`
2. Remove the `notion` block from `mcpServers`
3. Remove `'NOTION_TOKEN'` from `readSecrets()` in `src/container-runner.ts`
4. Remove `NOTION_TOKEN` from `.env`
5. Remove Notion sections from `groups/*/CLAUDE.md`
6. Rebuild:
   ```bash
   npm run build
   launchctl kickstart -k gui/$(id -u)/com.nanoclaw
   ```
