# Agent Looker - Claude Cowork Plugin

A plugin for [Claude Cowork](https://claude.com/docs/cowork/guide/plugins) that protects AI agents from unsafe URLs, malicious content, and prompt injection attacks via the [Agent Looker](https://agentlooker.ai/) MCP server.

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

> **Note:** Unlike the full Claude Code plugin, hooks in this Cowork edition do not make direct API calls — they inject security rules that guide Claude to use the MCP skills. All actual threat detection goes through the skills. Which hook events Cowork runs is not documented by Anthropic, so the skills are written to work even if the hooks never fire.

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

- Claude Desktop with [Claude Cowork](https://claude.com/docs/cowork/guide/plugins) and plugin support
- An Agent Looker account (sign up at the [dashboard](https://app.agentlooker.ai/))

No Node.js, no CLI, no setup script. Everything is installed from the Cowork UI.

## Installation

### 1. Add the marketplace

In Claude Desktop open **Customize → Plugins → Add marketplace** and enter:

```
Gogolook-Inc/agent-looker-claude-cowork
```

### 2. Install the plugin

Pick **agent-looker-for-claude-cowork** from the marketplace and install it. This registers the MCP connector, the hooks, and the four skills in one step.

### 3. Sign in to the connector

Open the plugin's **Connectors** tab and sign in to **agent-looker**. Cowork starts a standard OAuth 2.1 flow: a browser tab opens, you sign in with Google, and approve access. No token to copy. The resulting credential shows up in the dashboard under Tokens, named after the OAuth client, and can be revoked there.

### 4. Restart the session

Start a new Cowork session to activate the hooks and skills.

## Pointing at another environment (staging / develop)

Cowork has no environment variables, no settings file, and no way to edit a connector's URL, so the environment is baked into the plugin you install. Each git branch of this repo carries its own `.mcp.json`, and the marketplace lists one entry per branch:

| Marketplace entry | Branch | API |
|---|---|---|
| `agent-looker-for-claude-cowork` | `production` (default) | `https://api.agentlooker.ai/mcp` |
| `agent-looker-for-claude-cowork-staging` | `staging` | `https://api-staging.agentlooker.ai/mcp` |
| `agent-looker-for-claude-cowork-develop` | `develop` | `https://api-develop.agentlooker.ai/mcp` |

Install exactly one of them. They all register an MCP server named `agent-looker`, so installing two side by side will collide. To switch, uninstall the current one and install another.

The staging and develop entries are for internal testing. Their sign-in page sits behind HTTP Basic Auth at the CDN; the browser will prompt for it once before the Google login. The machine-to-machine OAuth endpoints (`/oauth/register`, `/oauth/token`, `/mcp`) are exempt, so the flow completes normally after that.

Maintainers: `.mcp.json` is the only file that differs between branches, and you never edit it by hand. The [mcp-url workflow](.github/workflows/mcp-url.yml) fails a pull request whose `.mcp.json` does not match the target branch, and on every push it rewrites the file to that branch's API and commits the fix. Promoting develop → staging → production therefore cannot carry the wrong URL upward.

## Project structure

```
.claude-plugin/
  plugin.json          # Plugin metadata
  marketplace.json     # Marketplace listing: one entry per environment
.mcp.json              # MCP connector; URL differs per branch
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

| Feature | [Claude Code plugin](https://github.com/Gogolook-Inc/agent-looker-claude-code) | Claude Cowork plugin |
|---------|-------------------|---------------------|
| PreToolUse URL blocking | Calls API directly, blocks before fetch | Injects rules; Claude calls MCP skill |
| PostToolUse content scan | Calls API directly, warns via context | Injects rules; Claude calls MCP skill |
| Install | `claude plugin` CLI + `bin/setup.mjs` | Cowork UI only |
| Authentication | Device flow; token stored in `~/.claude/settings.json` `env`, named `claude-code-cli_<hostname>` | OAuth 2.1 sign-in from the Connectors tab |
| Switching environment | `AGENT_LOOKER_MCP_URL` env var / `--mcp-url` | Install the matching marketplace entry |
| Config storage | `~/.claude/` (shared with Claude Code CLI) | Claude Desktop's own storage (`Claude-3p`), separate from `~/.claude/` |
| Node.js required | Yes (for hook scripts) | No |

## License

GPL-3.0 -- see [LICENSE](LICENSE) for details.
