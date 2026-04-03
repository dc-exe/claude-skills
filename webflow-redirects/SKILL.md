---
name: webflow-redirects
description: >
  Set up, manage, and explain 301 redirects in Webflow using the official Webflow MCP server.
  Use this skill whenever the user mentions redirects, 301s, old URLs, broken links, URL structure changes,
  domain migrations, page reorganizations, wildcard redirects, capture groups, or anything related to
  routing traffic in Webflow — even if they don't say the word "redirect" explicitly. Also trigger when
  a user asks why visitors are hitting 404s after a site restructure, or how to preserve SEO when changing URLs.
---

# Webflow 301 Redirects Skill

This skill guides Claude through setting up and managing 301 redirects in Webflow using the official Webflow MCP server. It covers all redirect types: single page, folder/wildcard, domain-to-domain, and bulk imports.

---

## Prerequisites & Important Caveats

> **Enterprise Only**: The Webflow Data API redirect endpoints (`/v2/sites/:site_id/redirects`) require an **Enterprise workspace**. If the user is on a lower plan, fall back to CSV + manual import.

> **MCP Connection**: Ensure the Webflow MCP server is connected. If tools are unavailable, prompt the user to connect via Claude's connector settings, then authenticate via OAuth.

> **Best practice limit**: Recommend a maximum of 1,000 redirect rules. Encourage wildcard redirects where possible to keep the count low.

> **Publish required**: After any redirect changes (via API or UI), the site must be **published** for redirects to take effect.

---

## Workflow

### Step 0 — Gather redirect data

Ask the user:

> "Do you have a CSV file with your redirects, or would you like to enter the URLs individually?"
> - **I have a CSV** — ask them to upload it and parse the `path` and `targetPath` columns.
> - **I'll enter them individually** — ask them to list or describe the URLs they want to redirect.

Wait for their answer before proceeding.

---

### Step 1 — Understand the scenario
Ask what changed: a single page, a folder, a full domain, or many URLs at once? Skip if data is already clear from an uploaded CSV.

---

### Step 2 — Check for duplicate old paths (MANDATORY — do before anything else)

Scan ALL old paths and identify any **exact duplicates** — cases where the same old path appears more than once with a different `targetPath`.

Webflow will throw an error if a CSV contains duplicate old paths. They must be resolved before the CSV can be imported.

**When duplicates are found:**

List all the duplicates clearly, showing each instance and its destination. Then ask the user:

> "I found duplicate old paths in your data — Webflow won't accept these. For each one, would you like to choose which rule to keep, or should I decide?"

- **User wants to choose** — show each duplicate group and wait for their selection before proceeding.
- **Claude decides** — keep the last occurrence of each duplicate (most recently added intent), discard the earlier ones, and note which were removed in your response so the user can review.

Never silently discard duplicates. Always tell the user which rules were removed and why.

**Example:**
```
DUPLICATE FOUND — old path /products/old%-page appears twice:
  1. /products/old%-page → /product/new-page-a
  2. /products/old%-page → /product/new-page-b
Which should I keep, or should I decide?
```

---

### Step 3 — Check for wildcard opportunities (MANDATORY)

Before generating any rules, scan ALL old paths together and ask: **can any be combined into a wildcard rule?**

Look for a shared **complete path segment** prefix across multiple old URLs. If old paths share a common prefix AND new paths follow the same slug pattern, collapse them into ONE wildcard rule using a capture group `(.*)`.

**Rules for valid wildcards:**

**Rule 1 — Never break a word or segment to create a wildcard.**
The wildcard prefix must end at a `/` — never mid-word or mid-segment. The shared text must be a complete, meaningful path segment.

❌ WRONG — prefix cuts mid-word:
```
/2016/post-a, /2018/post-b, /2024/post-c
Shared text is "/20" — this is mid-word. Do NOT create /20%(.*).
Keep these as individual rules.
```

✅ CORRECT — prefix ends cleanly at a segment boundary:
```
/blog/food, /blog/travel, /blog/music
Shared segment is "/blog/" — clean boundary. Use /blog/(.*). ✅
```

✅ CORRECT — full folder name shared:
```
/experience/exhibitions/show-a, /experience/exhibitions/show-b
Shared segment is "/experience/exhibitions/" — use /experience/exhibitions/(.*). ✅
```

**Rule 2 — Only wildcard when destinations are consistent.**
Destinations must either all go to the same single URL, or all follow the same pattern (e.g. `/blog/%1`). If destinations are arbitrary and unrelated, keep individual rules.

**Rule 3 — When in doubt, keep individual rules.**
A wrong wildcard misdirects traffic. Only collapse when the shared prefix is a complete, meaningful path segment ending at a `/`.

---

### Step 4 — Rule ordering (MANDATORY)

**How Webflow processes redirects:** Webflow checks rules top-to-bottom and stops at the first match. This means more specific rules must appear **above** broader wildcard rules in Webflow's redirect list.

**How Finsweet Bulk Import affects order:** The [Finsweet Attributes Chrome Extension](https://chromewebstore.google.com/detail/mjfibgdpclkaemogkfadpbdfoinnejep) adds a Bulk Import button to Webflow that is strongly recommended over Webflow's native import. However, **Finsweet reverses the CSV order on upload** — the first row in the CSV ends up at the bottom of Webflow's list, and the last row ends up at the top.

This means the CSV order must be the **reverse** of the desired Webflow order:

- **In Webflow** (top = matched first): specific rules on top, broader wildcards below
- **In CSV** (for Finsweet upload): broader wildcards first, specific rules last

**CSV ordering rules:**
1. Wildcard rules come **first** in the CSV (they will end up at the bottom in Webflow, matched last)
2. Among wildcard rules, **broader wildcards come before more specific wildcards** in the CSV (so the more specific wildcard ends up above the broader one in Webflow)
3. Individual (specific) rules come **last** in the CSV (they will end up at the top in Webflow, matched first)

✅ CORRECT CSV ordering (Finsweet upload):
```
/collection/(.*),/collection/%1               ← broader wildcard first in CSV (lands lower in Webflow)
/collection/artist/(.*),/artist/%1            ← specific wildcard next in CSV (lands above broader in Webflow)
/collection/specific%-page,/new-page          ← individual rule last in CSV (lands at top in Webflow) ✅
```

The resulting Webflow order (top to bottom) will be:
```
/collection/specific%-page → /new-page        ← matched first ✅
/collection/artist/(.*) → /artist/%1          ← matched second ✅
/collection/(.*) → /collection/%1             ← matched last ✅
```

❌ WRONG CSV ordering (Finsweet upload):
```
/collection/specific%-page,/new-page          ← lands at bottom in Webflow — never reached ✗
/collection/artist/(.*),/artist/%1            ← swallowed by broader wildcard above it ✗
/collection/(.*),/collection/%1               ← lands at top — catches everything ✗
```

---

### Step 5 — Apply escape characters (MANDATORY)

Before finalising any old path, escape ALL special characters using `%`. No exceptions.

| Character | Escaped form |
|---|---|
| `-` | `%-` |
| `?` | `%?` |
| `&` | `%&` |
| `=` | `%=` |
| `_` | `%_` |
| `+` | `%+` |
| `(` | `%(` |
| `)` | `%)` |
| `*` | `%*` |
| `%` | `%%` |

> Escape characters are **only needed in the old path** (`path` column) — never in `targetPath`.

---

### Step 6 — Generate the CSV (DEFAULT — always do this)

Produce a CSV with two columns: `path` and `targetPath`.

Ordering rules within the CSV (designed for Finsweet Bulk Import, which reverses order on upload):
1. **Broader wildcard rules first** (e.g. `/collection/(.*)`)
2. **More specific wildcard rules next** (e.g. `/collection/artist/(.*)`)
3. **Individual (specific) rules last** (e.g. `/collection/specific%-page`)

The CSV is the safe default — it works for all Webflow plans.

---

### Step 7 — After delivering the CSV, ask about Enterprise / MCP

Once the CSV is ready, ask:

> "Are you on a Webflow Enterprise plan? If so, I can connect to your site via MCP and apply these redirects directly — no manual importing needed."

- **Yes, Enterprise** — use MCP tools to create redirects via the API (see MCP Tool Usage section). Remind the user to publish after.
- **No / unsure** — recommend the Finsweet extension and give import instructions (see Bulk Import section below).

---

### Step 8 — Suggest testing
Tell the user to open each old URL in a new browser tab to verify it redirects correctly.

---

## Redirect Types Reference

### 1. Single Page Redirect
Use when one specific URL needs to point to a new URL.

```
path:        /old%-page
targetPath:  /new-page
```

**Note**: If the old URL is a currently live static page, the user must first delete it, save it as a draft, or change its slug. For CMS collection items, they must archive, delete, unpublish, save as draft, or change the slug first.

---

### 2. Folder / Wildcard Redirect
Use when redirecting many pages under one path structure. Capture groups use `(.*)` in the old path and `%1`, `%2`, etc. in the new path.

**Entire folder to one destination:**
```
path:        /old%-folder/(.*)
targetPath:  /new-destination
```

**Folder contents to matching new folder:**
```
path:        /old%-folder/(.*)
targetPath:  /new-folder/%1
```

**Multiple capture groups:**
```
path:        /blogs/(.*)/(.*) 
targetPath:  /articles/%1/%2
```

**Query string variables (`?category=food&post=pie` → `/blog/food/pie`):**
```
path:        /%?category%=(.*)%&post%=(.*)
targetPath:  /blog/%1/%2
```

---

### 3. Domain-to-Domain Redirect
Use when migrating from one domain to another entirely. **UI only — not available via API.**

1. Connect both old and new domains to the Webflow site
2. Set the new domain as the **default**
3. Publish the site

---

### 4. Bulk Redirects (CSV)
**CSV bulk import is UI only — not available via the API or MCP.**

#### Recommended: Finsweet Attributes Extension (Chrome)

Install the [Finsweet Attributes Chrome Extension](https://chromewebstore.google.com/detail/mjfibgdpclkaemogkfadpbdfoinnejep) before importing. It adds a **Bulk Import** button to Webflow's redirect settings and is strongly preferred over Webflow's native import for one critical reason:

> **Webflow's native Bulk Import replaces your entire redirect list.** To add new redirects without losing existing ones, you'd have to export your current list, merge in the new entries, and re-import the whole thing. The Finsweet extension lets you import additional redirects on top of your existing list without affecting anything already there.

**⚠️ Important — Finsweet reverses CSV order on upload.** The first row in your CSV ends up at the *bottom* of Webflow's list. See Step 4 for how to order your CSV correctly to account for this.

**Import steps:**
1. Install the [Finsweet Attributes Chrome Extension](https://chromewebstore.google.com/detail/mjfibgdpclkaemogkfadpbdfoinnejep)
2. Go to **Site Settings → Publishing tab** in Webflow
3. Click the **"Bulk Import"** button (added by the extension)
4. Upload the CSV (columns: `path`, `targetPath`)
5. **Publish the site**

---

## MCP Tool Usage (Enterprise only)

### List existing redirects
Use `data_sites_tool` to fetch the site ID, then:
```
GET /v2/sites/:site_id/redirects
```

### Create a redirect
```
POST /v2/sites/:site_id/redirects
Body: { "fromUrl": "/old-path", "toUrl": "/new-path" }
```

### Update a redirect
```
PATCH /v2/sites/:site_id/redirects/:redirect_id
Body: { "fromUrl": "/old-path", "toUrl": "/updated-path" }
```

### Delete a redirect
```
DELETE /v2/sites/:site_id/redirects/:redirect_id
```

**Required scope**: `sites:write`

---

## Manual UI Instructions (Fallback)

Use when MCP is unavailable, user is not on Enterprise, or for operations the API doesn't support.

### Add a single redirect:
1. Go to **Site Settings → Publishing → 301 Redirects**
2. Enter old URL in **Old path** field
3. Enter new URL in **Redirect to path** field
4. Click **Add redirect path**
5. **Publish your site**

### Delete a redirect:
1. Go to **Site Settings → Publishing → 301 Redirects**
2. Click the **trash icon** next to the redirect
3. **Publish your site**

---

## Common Scenarios & Example Outputs

| Scenario | path (old) | targetPath (new) |
|---|---|---|
| Single page moved | `/about%-us` | `/about` |
| Blog folder restructured | `/blog/(.*)` | `/articles/%1` |
| Category + post query params | `/%?category%=(.*)%&post%=(.*)` | `/blog/%1/%2` |
| All collection pages to one | `/products/(.*)` | `/shop` |
| Year-based blog URLs | Keep as individual rules — `/2021/(.*)`, `/2022/(.*)` etc. **do NOT** collapse to `/20%(.*)`  | `/blog` |
| Domain redirect | UI only | — |

---

## Localization Note

301 redirects are relative to the root domain and **do not apply to localized slugs**. If both `/old-url` and `/es/old-url` need redirecting, set them up as two separate redirect rules.

---

## After Making Changes

Always remind the user to:
1. ✅ **Publish the site** — redirects don't go live until published
2. ✅ **Test** — open each old URL in a new browser tab
3. ✅ **Check redirect count** — keep under 1,000; use wildcards where appropriate