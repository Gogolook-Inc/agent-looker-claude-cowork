# Agent Looker - Claude Cowork Plugin

A plugin for [Claude Cowork](https://claude.ai/code) that protects AI agents from unsafe URLs, malicious content, and prompt injection attacks via the [Agent Looker](https://agent-looker.whoscall.com/) MCP server.

## What it does

Agent Looker adds two layers of protection to your Claude Cowork sessions:

**Hooks (automatic)** -- System-level guards that inject security rules before and after web tool calls:

- **PreToolUse** -- Before every tool call, security rules are injected into Claude's context, ensuring it knows to call `check_url_safety` before any URL access (WebFetch, Bash curl/wget, etc.).
- **PostToolUse** -- After every tool call, security rules are re-injected so Claude knows to call `check_text_safety` on any external content received.

**Skills (Claude-driven)** -- Four skills that teach Claude when and how to call Agent Looker's MCP tools:

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `check-url-safety` | Before accessing any URL (curl, wget, git clone, etc.) | Claude calls `check_url_safety` before every URL access |
| `check-text-safety` | When processing external text from any source | Claude calls `check_text_safety` on all received external content |
| `report-risk-url` | Proactively, when a suspicious URL is discovered | Phishing, malware, scam, suspicious redirects |
| `report-risk-text` | Proactively, when suspicious text is discovered | Prompt injection, jailbreak, data leaks |

> **Note:** Unlike the full Claude Code plugin, hooks in this Cowork edition do not make direct API calls — they inject security rules that guide Claude to use the MCP skills. All actual threat detection goes through the skills.

## How protection works

```
WebFetch(url)
      |
      v
PreToolUse hook: injects security rules into Claude's context
      |
      Claude calls check_url_safety (MCP skill)
      |
      +-- UNSAFE --> Claude blocks the fetch and informs the user
      |
      +-- SAFE   --> WebFetch executes
                        |
                        v
               PostToolUse hook: injects security rules into Claude's context
                        |
                        Claude calls check_text_safety (MCP skill)
                        |
                        +-- BLOCK/FLAG --> Claude warns the user
                        +-- ALLOW     --> pass through
```

A safe URL can still serve malicious content. URL checks and content checks are two independent layers.

## Requirements

- [Claude Cowork](https://claude.ai/code) with MCP support
- Agent Looker MCP server configured in your Cowork workspace
- An Agent Looker account (sign up at the [dashboard](https://agent-looker.whoscall.com/))

## Installation

### 1. Configure the Agent Looker MCP server

Add the Agent Looker MCP server to your Claude Cowork workspace settings. The MCP server provides the `check_url_safety`, `check_text_safety`, `report_risk_url`, and `report_risk_text` tools that skills call into.

Refer to your Agent Looker dashboard for the MCP server URL and authentication token.

### 2. Load hooks

Copy `hooks/hooks.json` into your Claude Cowork hooks configuration, or reference it from your workspace's hook settings. The hooks inject Agent Looker security rules into Claude's context before and after `WebFetch` and `WebSearch` calls.

### 3. Load skills

Copy the `skills/` directory into your Claude Cowork skills directory. Each subdirectory contains a `SKILL.md` that teaches Claude when and how to call the corresponding MCP tool.

### 4. Restart Claude Cowork

Restart your Claude Cowork session to activate the hooks and skills.

## Project structure

```
hooks/
  hooks.json           # PreToolUse / PostToolUse hook definitions
                       # (injects security rules via additionalContext)
skills/
  check-url-safety/    # Skill: check URLs before access
  check-text-safety/   # Skill: check text content safety
  report-risk-url/     # Skill: report suspicious URLs
  report-risk-text/    # Skill: report suspicious text
```

## Difference from the full Claude Code plugin

| Feature | Claude Code plugin | Claude Cowork plugin |
|---------|-------------------|---------------------|
| PreToolUse URL blocking | Calls API directly, blocks before fetch | Injects rules; Claude calls MCP skill |
| PostToolUse content scan | Calls API directly, warns via context | Injects rules; Claude calls MCP skill |
| Setup script | Yes (`bin/setup.mjs`) | No |
| Authentication | `~/.agent-looker.cfg` | Via MCP server configuration |
| Node.js required | Yes (for hook scripts) | No |

## License

GPL-3.0 -- see [LICENSE](LICENSE) for details.
