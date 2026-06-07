# Claude Code Tool System

> Every built-in tool Claude Code can call, sorted by what it does.
> 42 tools across 8 categories. 🔒 = requires explicit enablement.
> Source: Claude Code internals analysis. Snapshot — may drift with platform releases.

## File Operations (6 tools)

| Tool | Purpose |
|------|---------|
| FileRead | Read file contents |
| FileEdit | Edit existing files |
| FileWrite | Write new files |
| Glob | Pattern-based file search |
| Grep | Content search (ripgrep) |
| NotebookEdit | Edit Jupyter notebooks |

## Execution (3 tools)

| Tool | Purpose |
|------|---------|
| Bash | Shell command execution |
| PowerShell | Windows shell execution |
| REPL | Interactive language runtime |

## Search & Fetch (3 tools)

| Tool | Purpose |
|------|---------|
| WebFetch | Fetch URL content |
| WebSearch | Web search |
| ToolSearch | Search deferred tool schemas |

## Agents & Tasks (10 tools)

| Tool | Purpose |
|------|---------|
| Agent | Spawn subagent (shadow clone) |
| SendMessage | Message a running agent |
| TaskCreate | Create a task |
| TaskGet | Get task status |
| TaskList | List tasks |
| TaskUpdate | Update task status |
| TaskStop | Stop a task |
| TaskOutput | Get task output |
| TeamCreate | Create agent team |
| TeamDelete | Delete agent team |

## Planning (4 tools)

| Tool | Purpose |
|------|---------|
| EnterPlanMode | Enter plan mode (read-only) |
| ExitPlanMode | Exit plan mode |
| EnterWorktree | Enter isolated git worktree |
| ExitWorktree | Exit worktree |

## MCP (4 tools)

| Tool | Purpose |
|------|---------|
| mcp | Call MCP server tool |
| ListMcpResources | List available MCP resources |
| ReadMcpResource | Read an MCP resource |
| McpAuth | Authenticate to MCP server |

## System (8 tools)

| Tool | Purpose |
|------|---------|
| AskUserQuestion | Prompt user for input |
| TodoWrite | Write plan/todo list (progress anchor) |
| Skill | Load and invoke a skill |
| Config | Read/write configuration |
| RemoteTrigger 🔒 | Trigger remote execution |
| CronCreate 🔒 | Create scheduled task |
| CronDelete 🔒 | Delete scheduled task |
| CronList 🔒 | List scheduled tasks |

## Experimental (4 tools)

| Tool | Purpose |
|------|---------|
| Sleep | Pause execution |
| Brief | Generate brief summary |
| StructuredOutput 🔒 | Structured output generation |
| LSP 🔒 | Language Server Protocol integration |
