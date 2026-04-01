# webflow-redirects

A Claude skill for generating and managing 301 redirects in Webflow — built on the [official Webflow documentation](https://help.webflow.com/hc/en-us/articles/33961294898835-How-do-I-set-up-redirects-in-Webflow).

---

## What it does

Setting up redirects in Webflow by hand is tedious and error-prone. This skill turns Claude into a redirect specialist that:

- Accepts a CSV of old/new URLs **or** individual URLs pasted directly into chat
- Automatically detects wildcard opportunities to consolidate multiple rules into one
- Enforces correct rule ordering so specific rules always fire before wildcards
- Escapes all special characters in old paths as required by Webflow
- Outputs a ready-to-upload CSV with the correct `path` / `targetPath` columns
- Optionally applies redirects directly to your Webflow site via MCP (Enterprise plan required)

---

## How to use

### Option A — Generate a CSV (works on all plans)

1. Start a conversation with Claude
2. Upload your existing CSV of redirects, or paste old → new URL pairs directly
3. Claude generates a validated, ready-to-import CSV
4. In Webflow: **Site Settings → Publishing tab → Bulk Import**
5. Upload the CSV and publish your site

### Option B — Apply via MCP (Enterprise only)

If you're on a Webflow Enterprise plan, Claude can connect to your site via the [official Webflow MCP server](https://developers.webflow.com/mcp/reference/overview) and apply redirects directly — no CSV importing needed.

---

## Installation

### Claude.ai

1. Download `webflow-redirects.skill`
2. Go to **Settings → Customize → Skills**
3. Upload the `.skill` file and toggle it on

### Claude Code

1. Unzip `webflow-redirects.skill`
2. Copy the `webflow-redirects/` folder to your skills directory:

```bash
# Global — available in all projects
cp -r webflow-redirects/ ~/.claude/skills/

# Project-specific
cp -r webflow-redirects/ .claude/skills/
```

Claude Code picks it up automatically. You can also trigger it explicitly with `/webflow-redirects`.

---

## CSV format

| Column | Description |
|---|---|
| `path` | Old URL path (relative, with special characters escaped) |
| `targetPath` | New URL path (relative, no escaping needed) |

**Example:**

```csv
path,targetPath
/old%-page,/new-page
/blog/(.*),/articles/%1
/experience/exhibitions/specific%-show,/exhibitions/specific-show
/experience/exhibitions/(.*),/exhibitions/%1
```

---

## Key behaviours

**Wildcard consolidation** — When multiple old paths share a complete path segment prefix (e.g. `/blog/post-a`, `/blog/post-b`), the skill collapses them into a single wildcard rule (`/blog/(.*)`). It never breaks mid-word to create a wildcard.

**Rule ordering** — Specific rules are always placed before their wildcard siblings in the output CSV, so Webflow matches them in the right order.

**Automatic escaping** — Special characters (`-`, `?`, `&`, `=`, `_`, `*`, etc.) are escaped in all old paths automatically.

**Graceful fallback** — If MCP isn't available or the user isn't on Enterprise, the skill falls back to CSV generation with manual import instructions.

---

## Webflow plan requirements

| Feature | Free / Basic / CMS / Business | Enterprise |
|---|---|---|
| CSV bulk import | ✅ | ✅ |
| Apply via MCP / API | ❌ | ✅ |

---

## Contributing

Found an edge case? Open an issue or submit a PR. The skill is a single `SKILL.md` file — easy to read and modify.

---

## Based on

[How do I set up redirects in Webflow?](https://help.webflow.com/hc/en-us/articles/33961294898835-How-do-I-set-up-redirects-in-Webflow) — Official Webflow Help Center
