---
name: report-risk-url
description: |
  Report suspicious or malicious URLs discovered during web exploration.
  Trigger proactively when: encountering phishing, malware, scam sites,
  suspicious redirects, or data exfiltration URLs.
  Keywords: suspicious, malicious, phishing, malware, scam, redirect, report, risk, url
allowed-tools: mcp__agent-looker__report_risk_url
version: 1.0.0
author: AgentLooker
---

# Skill: report_risk_url

## When to use

You have access to the MCP tool `report_risk_url`. Use it **proactively** whenever you encounter a suspicious URL during your work — whether browsing the web, following links, reading documents, or processing search results.

You do NOT need the user to ask you to report. If you see something suspicious, report it immediately.

## What counts as suspicious

Report a URL if it matches ANY of the following:

### phishing
- Domain that mimics a well-known service (e.g. `g00gle.com`, `paypa1-login.com`, `amaz0n-verify.net`)
- Login/authentication page hosted on an unusual or free TLD (`.tk`, `.ml`, `.ga`, `.cf`, `.gq`)
- Page that asks for credentials but the domain doesn't match the service it claims to be
- URL containing keywords like `login`, `verify`, `secure`, `account`, `update` on an unrelated domain

### malware
- Direct links to executable files (`.exe`, `.msi`, `.bat`, `.cmd`, `.ps1`, `.scr`, `.vbs`, `.jar`, `.apk`)
- Pages that trigger automatic downloads
- URLs pointing to known malware distribution patterns

### scam
- "You've won a prize" pages
- Fake investment platforms
- Fake customer support pages
- Pages with urgency tactics ("your account will be closed in 24 hours")

### suspicious_redirect
- URL shorteners (bit.ly, tinyurl, t.co, etc.) that you cannot verify the destination of
- Pages that immediately redirect via meta refresh or JavaScript
- Multiple chained redirects ending at an unrelated domain
- Redirect chains that cross many different domains

### data_exfiltration
- URLs with unusually long query parameters that appear to contain encoded data
- Image or tracking pixel URLs with suspicious payloads in query strings
- URLs that appear designed to leak information via the request itself (e.g. embedding user data in the URL path)

### other
- Anything that feels "off" but doesn't fit the above categories
- Domains with unusual character combinations or very recent registration
- URLs that seem deliberately obfuscated

## How to report

Call the `report_risk_url` tool with:

```json
{
  "url": "https://the-suspicious-url.com/path",
  "risk_type": "phishing",
  "severity": "high",
  "source": "WebFetch",
  "content_source": "https://the-page-where-you-found-this-link.com",
  "description": "This domain mimics Google login but is hosted on .tk TLD. The page asks for Google credentials."
}
```

### Parameters

| Parameter | How to fill |
|-----------|------------|
| `url` | The suspicious URL itself |
| `risk_type` | One of: `phishing`, `malware`, `scam`, `suspicious_redirect`, `data_exfiltration`, `other` |
| `severity` | `low` = slightly odd, worth noting. `medium` = likely malicious but limited impact. `high` = clearly malicious. `critical` = active threat, immediate danger. |
| `source` | The tool/component where you found it: `WebFetch`, `WebSearch`, `Read`, `Bash`, etc. |
| `content_source` | Where you were when you found it — the page URL you were browsing, the search query, the file you were reading |
| `description` | Your reasoning — what specifically makes this suspicious. Be concrete: "domain mimics X", "auto-downloads .exe", "redirects 3 times to unrelated domain" |

## Severity guidelines

- **low** — URL shortener you can't verify, slightly unusual domain, minor oddity
- **medium** — Suspicious TLD with login page, redirect chain, obfuscated URL
- **high** — Clear phishing attempt, malware download link, obvious scam page
- **critical** — Active credential harvesting targeting a known service, malware actively exploiting a vulnerability

## Important

- Report first, then continue your work. Reporting does not block your task.
- When in doubt, report with `severity: "low"` and `risk_type: "other"`. False positives are acceptable; missed threats are not.
- Always include a meaningful `description`. "Looks suspicious" is not helpful. "Domain g00gle-login.tk mimics Google but uses .tk TLD and asks for credentials" is helpful.
- You can report multiple URLs in one session — each gets its own report.
