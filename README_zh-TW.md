# Agent Looker - Claude Cowork Plugin

透過 [Agent Looker](https://agent-looker.whoscall.com/) MCP server，保護你的 [Claude Cowork](https://claude.ai/code) AI agent 免於不安全的 URL、惡意內容和 prompt injection 攻擊。

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

- 支援 MCP 的 [Claude Cowork](https://claude.ai/code)
- 在你的 Cowork workspace 中設定 Agent Looker MCP server
- Agent Looker 帳號（在 [dashboard](https://agent-looker.whoscall.com/) 註冊）

## 安裝

### 1. 設定 Agent Looker MCP server

在 Claude Cowork workspace 設定中新增 Agent Looker MCP server。MCP server 提供 skills 所需的 `check_url_safety`、`check_text_safety`、`report_risk_url` 和 `report_risk_text` tools。

MCP server URL 和認證 token 請參考你的 Agent Looker dashboard。

### 2. 載入 hooks

將 `hooks/hooks.json` 複製到 Claude Cowork 的 hook 設定中，或在 workspace hook 設定中引用它。Hooks 會在 `WebFetch` 和 `WebSearch` 呼叫前後，將 Agent Looker 安全規則注入 Claude 的 context。

### 3. 載入 skills

將 `skills/` 目錄複製到 Claude Cowork 的 skills 目錄。每個子目錄包含一個 `SKILL.md`，教導 Claude 何時以及如何呼叫對應的 MCP tool。

### 4. 重新啟動 Claude Cowork

重新啟動 Claude Cowork 對話以啟用 hooks 和 skills。

## 專案結構

```
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

| 功能 | Claude Code plugin | Claude Cowork plugin |
|------|-------------------|---------------------|
| PreToolUse URL 攔截 | 直接呼叫 API，在 fetch 前阻擋 | 注入規則；由 Claude 呼叫 MCP skill |
| PostToolUse 內容掃描 | 直接呼叫 API，透過 context 警告 | 注入規則；由 Claude 呼叫 MCP skill |
| Setup script | 有（`bin/setup.mjs`） | 無 |
| 認證方式 | `~/.agent-looker.cfg` | 透過 MCP server 設定 |
| 需要 Node.js | 是（用於 hook scripts） | 否 |

## 授權條款

GPL-3.0——詳見 [LICENSE](LICENSE)。
