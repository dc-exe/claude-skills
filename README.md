# webflow-redirects

A Claude skill for generating and managing 301 redirects in Webflow — built on the [official Webflow documentation](https://help.webflow.com/hc/en-us/articles/33961294898835-How-do-I-set-up-redirects-in-Webflow).

---

## What it does

Setting up redirects in Webflow by hand is tedious and error-prone. This skill turns Claude into a redirect specialist that:

- Accepts a CSV of old/new URLs **or** individual URLs pasted directly into chat
- Detects and flags duplicate old paths before they cause Webflow import errors
- Automatically checks for wildcard opportunities to consolidate multiple rules into one
- Enforces correct rule ordering for Webflow's top-to-bottom matching logic
- Escapes all special characters in old paths as required by Webflow
- Outputs a ready-to-upload CSV with the correct `path` / `targetPath` columns, ordered for [Finsweet Bulk Import](#recommended-finsweet-attributes-extension)
- Optionally applies redirects directly to your Webflow site via MCP (Enterprise plan required)

---

## How to use

### Option A — Generate a CSV (works on all plans)

1. Start a conversation with Claude
2. Upload your existing CSV of redirects, or paste old → new URL pairs directly
3. Claude validates the data (duplicates, wildcards, escaping, ordering) and generates a ready-to-import CSV
4. Import using the [Finsweet extension](#recommended-finsweet-attributes-extension) or Webflow's native Bulk Import
5. Publish your site

### Option B — Apply via MCP (Enterprise only)

If you're on a Webflow Enterprise plan, Claude can connect to your site via the [official Webflow MCP server](https://developers.webflow.com/mcp/reference/overview) and apply redirects directly — no CSV importing needed.

---

## Installation

### Claude.ai

1. Go to **Settings → Customize → Skills**
2. Click **Upload skill**
3. Upload the `SKILL.md` file from the `webflow-redirects/` folder
4. Toggle the skill on

### Claude Code

Clone the repo and copy the `webflow-redirects/` folder to your skills directory:

```bash
git clone https://github.com/dc-exe/claude-skills.git

# Global — available in all projects
cp -r claude-skills/webflow-redirects/ ~/.claude/skills/

# Project-specific
cp -r claude-skills/webflow-redirects/ .claude/skills/
```

Claude Code picks it up automatically at startup. You can also trigger it explicitly with `/webflow-redirects`.

---

## Recommended: Finsweet Attributes Extension

Install the **[Finsweet Attributes Chrome Extension](https://chromewebstore.google.com/detail/mjfibgdpclkaemogkfadpbdfoinnejep)** before importing your CSV.

It adds a **Bulk Import** button to Webflow's redirect settings and is strongly preferred over Webflow's native import:

> **Webflow's native Bulk Import replaces your entire redirect list.** To safely add new redirects without losing existing ones, you'd have to export your current list, merge in the new entries, and re-import everything. The Finsweet extension lets you import additional redirects on top of your existing list without touching anything already there.

### ⚠️ Important: Finsweet reverses CSV order on upload

The Finsweet extension uploads CSV rows in reverse — the **first row in your CSV ends up at the bottom** of Webflow's redirect list, and the last row ends up at the top.

Webflow matches redirects top-to-bottom and stops at the first match, so more specific rules must sit above broader wildcard rules. The CSV Claude generates accounts for this reversal automatically:

| Position in CSV | Lands in Webflow | Matched |
|---|---|---|
| Broader wildcards (first) | Bottom | Last |
| Specific wildcards (middle) | Middle | Second |
| Individual rules (last) | Top | First ✅ |

---

## CSV format

| Column | Description |
|---|---|
| `path` | Old URL path (relative, special characters escaped) |
| `targetPath` | New URL path (relative, no escaping needed) |

**Example — correct CSV order for Finsweet upload:**

```csv
path,targetPath
/collection/(.*),/collection/%1
/collection/artist/(.*),/artist/%1
/collection/specific%-page,/new-page
```

After Finsweet upload, Webflow's order (top to bottom) becomes:
```
/collection/specific-page     → /new-page         ← matched first ✅
/collection/artist/(.*)       → /artist/%1         ← matched second ✅
/collection/(.*)              → /collection/%1     ← matched last ✅
```

---

## Key behaviours

**Duplicate detection** — Scans all old paths for exact duplicates before generating any output. Webflow throws an error on duplicate old paths. Claude flags each one and asks whether you want to choose which to keep, or have it decide.

**Wildcard consolidation** — When multiple old paths share a complete path segment prefix, the skill collapses them into a single wildcard rule. It never breaks mid-word to create a wildcard — the prefix must end cleanly at a `/`.

**Rule ordering** — Produces CSV rows in the correct order for Finsweet's reversed upload: broader wildcards first, more specific wildcards next, individual rules last.

**Automatic escaping** — Special characters (`-`, `?`, `&`, `=`, `_`, `*`, etc.) are escaped in all old paths automatically. New paths are never escaped.

**Graceful fallback** — If MCP isn't available or the user isn't on Enterprise, the skill falls back to CSV generation with Finsweet import instructions.

---

## Webflow plan requirements

| Feature | Free / Basic / CMS / Business | Enterprise |
|---|---|---|
| CSV bulk import (Finsweet) | ✅ | ✅ |
| Apply via MCP / API | ❌ | ✅ |

---

## Contributing

Found an edge case? Open an issue or submit a PR. The skill is a single `SKILL.md` file — easy to read and modify.

---

## Based on

[How do I set up redirects in Webflow?](https://help.webflow.com/hc/en-us/articles/33961294898835-How-do-I-set-up-redirects-in-Webflow) — Official Webflow Help Center