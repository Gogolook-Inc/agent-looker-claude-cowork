# Agent Looker - Claude Cowork Plugin

透過 [Agent Looker](https://agentlooker.ai/) MCP server，保護你的 [Claude Cowork](https://claude.com/docs/cowork/guide/plugins) AI agent 免於不安全的 URL、惡意內容和 prompt injection 攻擊。

## 功能介紹

Agent Looker 為你的 Claude Cowork 對話加上兩層防護：

**Hooks（自動注入）**——在 web 工具呼叫前後自動注入安全規則：

- **PreToolUse**——每次 tool call 前，將安全規則注入 Claude 的 context，確保 Claude 知道要在任何 URL 存取前呼叫 `check_url_safety`（WebFetch、Bash curl/wget 等）。
- **PostToolUse**——每次 tool call 完成後，再次注入安全規則，提醒 Claude 對任何收到的外部內容呼叫 `check_text_safety`。

**Skills（Claude 主動驅動）**——四個 skill 教 Claude 何時以及如何呼叫 Agent Looker 的 MCP tools：

| Skill | 觸發時機 | 用途 |
|-------|---------|------|
| `check-url-safety` | 用任何方式存取 URL 前（curl、wget、git clone 等） | Claude 在每次存取 URL 前呼叫 `check_url_safety` |
| `check-text-safety` | 處理來自任何來源的外部文字時 | Claude 對所有收到的外部內容呼叫 `check_text_safety` |
| `report-risk-url` | 主動發現可疑 URL 時 | 釣魚、惡意軟體、詐騙、可疑重新導向 |
| `report-risk-text` | 主動發現可疑文字時 | Prompt injection、jailbreak、資料洩漏 |

> **注意：** 與完整的 Claude Code plugin 不同，此 Cowork 版本的 hooks 不會直接呼叫 API——它們注入安全規則來引導 Claude 使用 MCP skills。所有實際的威脅偵測都透過 skills 進行。

## 防護流程

```
WebFetch(url)
      |
      v
PreToolUse hook：將安全規則注入 Claude 的 context
      |
      Claude 呼叫 check_url_safety（MCP skill）
      |
      +-- 不安全 --> Claude 阻止 fetch 並通知使用者
      |
      +-- 安全   --> WebFetch 執行
                        |
                        v
               PostToolUse hook：將安全規則注入 Claude 的 context
                        |
                        Claude 呼叫 check_text_safety（MCP skill）
                        |
                        +-- BLOCK/FLAG --> Claude 警告使用者
                        +-- ALLOW     --> 正常通過
```

安全的 URL 仍然可能提供惡意內容。URL 檢查和內容檢查是兩層獨立的防護。

## 系統需求

- 支援 plugin 的 Claude Desktop 與 [Claude Cowork](https://claude.com/docs/cowork/guide/plugins)
- Agent Looker 帳號（在 [dashboard](https://app.agentlooker.ai/login) 註冊）

## 安裝

### 1. 加入 marketplace

在 Claude Desktop 開啟 **Customize → Plugins → Add marketplace**，輸入：

```
Gogolook-Inc/agent-looker-claude-cowork
```

### 2. 安裝 plugin

在 marketplace 中選擇 **agent-looker-for-claude-cowork** 安裝。這一步會同時註冊 MCP connector、hooks 和四個 skills。

### 3. 登入 connector

打開 plugin 的 **Connectors** 分頁，對 **agent-looker** 按下登入。Cowork 會走標準的 OAuth 2.1 流程：開啟瀏覽器、用 Google 帳號登入、同意授權即可，不需要複製任何 token。產生的憑證會出現在 dashboard 的 Tokens 頁，名稱取自 OAuth client，可在那裡撤銷。

### 4. 重新開啟對話

開啟新的 Cowork 對話以啟用 hooks 和 skills。

## 切換到其他環境（staging / develop）

這個 repo 每個 git branch 各自帶一份 `.mcp.json`，marketplace 對應列出一個 entry：

| Marketplace entry | Branch | API |
|---|---|---|
| `agent-looker-for-claude-cowork` | `production`（預設） | `https://api.agentlooker.ai/mcp` |
| `agent-looker-for-claude-cowork-staging` | `staging` | `https://api-staging.agentlooker.ai/mcp` |
| `agent-looker-for-claude-cowork-develop` | `develop` | `https://api-develop.agentlooker.ai/mcp` |

只能裝其中一個。三個 entry 註冊的 MCP server 都叫 `agent-looker`，同時裝兩個會互相衝突。要換環境，先解除安裝目前的再裝另一個。

> `.mcp.json` 是 branch 之間唯一不同的檔案，而且不需要手動改。[mcp-url workflow](.github/workflows/mcp-url.yml) 會讓 `.mcp.json` 與目標 branch 不符的 pull request 失敗，每次 push 也會把檔案改寫成該 branch 的 API 並自動 commit。

## 專案結構

```
.claude-plugin/
  plugin.json          # Plugin 後設資料
  marketplace.json     # Marketplace 列表：每個環境一個 entry
.mcp.json              # MCP connector；網址依 branch 不同
hooks/
  hooks.json           # PreToolUse / PostToolUse hook 定義
                       # （透過 additionalContext 注入安全規則）
skills/
  check-url-safety/    # Skill：存取前檢查 URL 安全性
  check-text-safety/   # Skill：檢查文字內容安全性
  report-risk-url/     # Skill：回報可疑 URL
  report-risk-text/    # Skill：回報可疑文字
```

## 與完整 Claude Code plugin 的差異

| 功能 | [Claude Code plugin](https://github.com/Gogolook-Inc/agent-looker-claude-code) | Claude Cowork plugin |
|------|-------------------|---------------------|
| PreToolUse URL 攔截 | 直接呼叫 API，在 fetch 前阻擋 | 注入規則；由 Claude 呼叫 MCP skill |
| PostToolUse 內容掃描 | 直接呼叫 API，透過 context 警告 | 注入規則；由 Claude 呼叫 MCP skill |
| 安裝方式 | `claude plugin` CLI + `bin/setup.mjs` | 只用 Cowork 介面 |
| 認證方式 | Device flow；token 存在 `~/.claude/settings.json` 的 `env`，名稱為 `claude-code-cli_<主機名稱>` | 從 Connectors 分頁走 OAuth 2.1 登入 |
| 切換環境 | `AGENT_LOOKER_MCP_URL` 環境變數 / `--mcp-url` | 安裝對應的 marketplace entry |
| 設定存放 | `~/.claude/`（與 Claude Code CLI 共用） | Claude Desktop 自己的儲存空間（`Claude-3p`），與 `~/.claude/` 分開 |
| 需要 Node.js | 是（用於 hook scripts） | 否 |

## 授權條款

GPL-3.0——詳見 [LICENSE](LICENSE)。
