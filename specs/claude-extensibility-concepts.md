# Claude Code Extensibility: Commands, Hooks, Plugins, Skills, and MCPs

> A reference guide to the five extensibility mechanisms in Claude Code and how they differ.

---

## Quick Comparison

| Concept | Who triggers it | When it runs | Where it's defined | Primary use |
|---------|----------------|--------------|-------------------|-------------|
| **Command** | User (explicitly) | On demand | `.claude/commands/*.md` | Custom reusable instructions |
| **Hook** | System (automatically) | On specific events | `settings.json` | Automated side effects |
| **Plugin** | N/A (installed) | Passively available | IDE / editor | Expands the editing environment |
| **Skill** | User (via Skill tool) | On demand | Agent system | Specialized, multi-step agents |
| **MCP** | Claude (via tool call) | During a conversation | MCP server config | External tools and data sources |

---

## Commands (Slash Commands)

**What they are:** Custom slash commands you define as Markdown files. When you type `/command-name` in Claude Code, the file's content becomes Claude's instruction set for that session.

**How to create one:**

Create a file at `.claude/commands/my-command.md`:

```markdown
---
description: Short description shown in the command picker
allowed-tools: Bash, Read, Edit
model: sonnet
---

# My Command

Instructions for Claude go here. You can include:
- Step-by-step workflows
- Specific constraints
- Output formats
```

**How to invoke:** Type `/my-command` in Claude Code.

**Examples in this repo:**

| Command | File | Purpose |
|---------|------|---------|
| `/EA-prime` | `.claude/commands/EA-prime.md` | Understand the codebase quickly |
| `/EA-handoff` | `.claude/commands/EA-handoff.md` | Save session state for later |
| `/EA-pickup` | `.claude/commands/EA-pickup.md` | Resume from a saved handoff |
| `/EA-install` | `.claude/commands/EA-install.md` | Install commands globally |

**Key characteristics:**
- User-initiated — Claude does nothing until you invoke the command
- Scoped to a project (`.claude/commands/`) or global (`~/.claude/commands/`)
- Can specify which tools Claude is allowed to use
- Can specify which model to use
- The Markdown content is the full prompt sent to Claude

---

## Hooks

**What they are:** Shell commands that run automatically when specific events occur in Claude Code. Unlike commands, you never invoke hooks manually — they fire on their own.

**Where they live:** Defined in `settings.json` (project or global).

**Available events:**

| Event | When it fires |
|-------|--------------|
| `PreToolUse` | Before Claude calls any tool |
| `PostToolUse` | After Claude finishes a tool call |
| `UserPromptSubmit` | When you submit a message to Claude |
| `Stop` | When Claude finishes responding |

**Example — auto-format after every file edit:**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $CLAUDE_TOOL_INPUT_FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

**Key characteristics:**
- Automatically triggered — no user action needed
- Run as shell commands on your local machine
- Can block Claude from proceeding (exit code non-zero = error)
- Receive context about the event via environment variables
- Useful for: linting, formatting, logging, safety checks, notifications

**Hooks vs Commands at a glance:**

```
Commands: User types /something → Claude runs instructions
Hooks:    Event fires automatically → Shell script runs
```

---

## Plugins

**What they are:** Installable extensions that integrate Claude Code into your development environment. Plugins are passive — they don't run instructions or scripts, they make Claude accessible from within tools you already use.

**Available plugins:**

| Plugin | Platform | What it adds |
|--------|----------|-------------|
| **VS Code Extension** | Visual Studio Code | Claude Code panel, inline suggestions, file context |
| **JetBrains Plugin** | IntelliJ, PyCharm, etc. | Claude Code integration in JetBrains IDEs |

**How they differ from commands and hooks:**

- A plugin is something you **install** into your editor, not something you write
- Plugins don't change how Claude reasons or what it can do
- Plugins provide a **surface** (UI, keybindings, context menus) to interact with Claude
- No Markdown files, no `settings.json` — installed via your editor's extension marketplace

**When to use:** When you want to use Claude Code without leaving your editor. The CLI remains the full-featured interface; plugins bring a subset of that experience into your IDE.

---

## Skills

**What they are:** Pre-built, specialized agents bundled with Claude Code (or a Claude Code environment) that handle complex, multi-step tasks. Skills are more sophisticated than commands — they have their own logic, can spawn sub-agents, and are maintained by the platform rather than defined by you.

**How to invoke:** Type `/skill-name` in Claude Code, or Claude invokes them internally via the `Skill` tool.

**Examples of built-in skills:**

| Skill | What it does |
|-------|-------------|
| `review` | Reviews a pull request with structured feedback |
| `security-review` | Audits pending branch changes for security issues |
| `init` | Generates a `CLAUDE.md` for a new codebase |
| `update-config` | Modifies `settings.json` (hooks, permissions, env vars) |
| `claude-api` | Builds and debugs Anthropic SDK apps |
| `schedule` | Creates and manages scheduled remote agents |

**How skills differ from commands:**

| | Commands | Skills |
|--|---------|--------|
| **Defined by** | You (Markdown files) | Platform / system |
| **Complexity** | Simple to moderate | Complex, multi-step |
| **Logic** | Declarative (Markdown prompt) | Procedural (can use sub-agents) |
| **Customizable** | Yes — edit the `.md` file | No — provided as-is |
| **Scope** | Your project or global config | Available system-wide |

**Key characteristics:**
- Built-in — no files to create or configure
- Can orchestrate multiple agents working in parallel
- Optimized for specific recurring workflows
- Listed in the system context so Claude knows when to trigger them

---

## MCPs (Model Context Protocol)

**What they are:** Standardized servers that give Claude access to external tools, data sources, and services. An MCP server exposes a set of "tools" (functions Claude can call) and optionally "resources" (data Claude can read). Claude calls these tools mid-conversation, exactly like it calls built-in tools (Read, Edit, Bash).

**How they work:**

```
User asks Claude something
  → Claude decides it needs external data/action
    → Claude calls an MCP tool (e.g., mcp__github__create_issue)
      → MCP server executes the call against the real service
        → Result returned to Claude
          → Claude continues the conversation
```

**Where they're configured:** In `settings.json` under `mcpServers`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

**Common MCP servers:**

| Server | What it provides |
|--------|----------------|
| GitHub MCP | Create issues, PRs, read repos, manage branches |
| Filesystem MCP | Extended file operations beyond built-in tools |
| Postgres MCP | Query and mutate a Postgres database |
| Slack MCP | Send messages, read channels |
| Browser MCP | Automate web browsers |

**Key characteristics:**
- Claude-initiated — the user doesn't call MCPs directly; Claude decides when to use them
- Run as separate processes (local or remote)
- Communicate via a standard protocol (JSON-RPC over stdio or HTTP)
- Tools appear in Claude's tool list just like built-in tools
- Resources provide read-only context (files, docs, database rows)
- Require explicit configuration and usually credentials/tokens

**MCPs vs the others:**

```
Commands → You tell Claude what to do
Hooks    → Events trigger your shell scripts
Plugins  → Your editor gains Claude UI
Skills   → Platform-provided complex agents
MCPs     → Claude gains access to external services
```

---

## Putting It All Together

Here's how all five concepts coexist in a typical Claude Code setup:

```
┌─────────────────────────────────────────────────────┐
│ DEVELOPER ENVIRONMENT                               │
│                                                     │
│  ┌──────────────┐    ┌──────────────────────────┐   │
│  │ VS Code      │    │ Claude Code CLI          │   │
│  │ (Plugin)     │◄──►│                          │   │
│  └──────────────┘    │  /EA-prime  (Command)    │   │
│                      │  /review    (Skill)      │   │
│                      │                          │   │
│  Auto-formatting     │  settings.json           │   │
│  on every edit ◄─────┤  (Hooks: PostToolUse)    │   │
│  (Hook)              │                          │   │
│                      └──────────┬───────────────┘   │
└─────────────────────────────────┼───────────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │ External Services (MCPs)   │
                    │  • GitHub API              │
                    │  • Database               │
                    │  • Slack                  │
                    └────────────────────────────┘
```

### Decision guide

- **"I want to give Claude a reusable workflow I can trigger on demand"** → **Command**
- **"I want something to happen automatically whenever Claude edits a file"** → **Hook**
- **"I want Claude in my IDE without switching to the terminal"** → **Plugin**
- **"I want a sophisticated, multi-step agent for a specific recurring task"** → **Skill**
- **"I want Claude to be able to interact with an external service"** → **MCP**

---

*See also: [Claude in Issues](../guides/claude-in-issues.md) · [Claude in PRs](../guides/claude-in-prs.md) · [Mobile Workflow](../guides/mobile-workflow.md)*
