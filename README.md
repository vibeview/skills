# VibeView agent skills

Skills and an MCP server that let a coding agent verify its work on a **live cloud device**: upload a build, start an iOS, Android, Apple TV or Android TV session, drive the app by UI-tree refs, read logs, take screenshots, and stop when only a human can act.

[VibeView](https://vibeview.io) streams iOS simulators, Android emulators, Apple TV and Android TV simulators, and Roku devices (beta) to a browser and to agents. No Mac, no local simulator, no device on the desk. Docs: [AI agent control](https://vibeview.io/docs/agent-control).

## Install

You need the `vibeview` CLI, logged in:

```bash
npm install -g vibeview
vibeview login
```

Then pick whichever fits your agent.

**Any agent that reads skills** (Claude Code, Cursor, Codex, and others via the `skills` CLI):

```bash
npx skills add vibeview/skills
```

**Claude Code, as a plugin** (installs the skill and registers the MCP server in one step):

```text
/plugin marketplace add vibeview/skills
/plugin install vibeview@vibeview
```

**Claude Code, from the CLI you already installed** (copies the skill into `~/.claude/skills/` and offers to register the MCP server):

```bash
vibeview agent install
```

**Any MCP client**: run `vibeview mcp` over stdio.

```json
{
  "mcpServers": {
    "vibeview": { "command": "vibeview", "args": ["mcp"] }
  }
}
```

With Claude Code specifically: `claude mcp add vibeview -- vibeview mcp`.

## What's here

| Path | What it is |
| --- | --- |
| `skills/vibeview-agent/SKILL.md` | The agent workflow: build requirements, the dev loop, verifying with `ui-tree`, screenshots and logs, TV d-pad navigation, the human hand-off rule, cleanup, and the full command reference for both the CLI and MCP. |
| `.claude-plugin/` | Claude Code plugin manifest: the skill plus the `vibeview mcp` server. |

## Where this comes from

The canonical copy of the skill ships inside the `vibeview` npm package (`vibeview agent install` copies it from there). This repository mirrors it so agents and registries can discover it on GitHub. Versions track the CLI.

Issues and questions: support@vibeview.io.
