---
name: check-url-safety
description: |
  Check URL safety before any form of access. Trigger when: about to visit,
  fetch, curl, clone, download, redirect to, or otherwise interact with a URL.
  Keywords: url, link, fetch, browse, visit, navigate, open, curl, wget, clone,
  download, redirect, safety, threat
allowed-tools: mcp__agent-looker__check_url_safety
version: 1.0.0
author: AgentLooker
---

# Skill: check_url_safety

## When to use

> **[PreToolUse equivalent]** In environments with hooks, URL safety is checked automatically before `WebFetch`. In this environment, **you are the hook** — you must call `check_url_safety` yourself before any URL access.

**Mandatory — call before:**

- **`WebFetch`** — every time, before the call, no exceptions
- `Bash` commands that access a URL: `curl`, `wget`, `git clone`, `npm install <url>`, `pip install <url>`, etc.
- Following any link found in a document, search result, error message, or external text
- Following a redirect to a new domain
- Opening or downloading a file from a URL
- Accessing an API endpoint provided by external content (not hardcoded in the project)
- Any URL the user provides and asks you to interact with
- Shortened URLs (bit.ly, t.co, etc.) — check before following

## How to use

Call the `check_url_safety` tool with:

```json
{
  "url": "https://example.com/page"
}
```

### Parameters

| Parameter | How to fill |
|-----------|------------|
| `url` | The full URL to check, including protocol (`https://` or `http://`) |

## Reading the result

The tool returns two pieces of content:

1. **Human-readable summary:**
   - `SAFE: "url" — no threats detected.` → URL is clean, safe to access
   - `UNSAFE: "url" matched the following threats: ...` → Do NOT access this URL

2. **JSON result:**
```json
{
  "url": "https://example.com",
  "isSafe": true,
  "threats": []
}
```

When `isSafe` is `false`, the `threats` array contains threat types:

| Threat | Meaning |
|--------|---------|
| `MALWARE` | Site distributes malicious software |
| `SOCIAL_ENGINEERING` | Phishing site that tricks users into revealing information |
| `UNWANTED_SOFTWARE` | Site distributes unwanted/deceptive software |
| `POTENTIALLY_HARMFUL_APPLICATION` | Site hosts potentially dangerous applications |

## What to do with the result

- **SAFE** → Proceed normally.
- **UNSAFE** → Do NOT access the URL by any method. Inform the user of the threat type. If the URL came from external content (not directly from the user), also consider filing a `report_risk_url` report.

## Important

- Always check before accessing unknown URLs — regardless of the tool or method you plan to use.
- If the check fails (network error, service unavailable), inform the user and proceed with caution.
- This tool checks against known threat databases. It does NOT detect zero-day threats or brand-new phishing sites. If a URL passes the check but still looks suspicious, trust your judgment and report it via `report_risk_url`.
