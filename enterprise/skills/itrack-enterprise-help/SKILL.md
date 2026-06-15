---
name: itrack-enterprise-help
description: >
  Route ITrack Enterprise user questions to the correct help documentation.
  Use this skill whenever a user asks how to do something in ITrack Enterprise,
  asks about a feature, workflow, screen, or setting, or asks where to find
  something in the application. Triggers include questions like "how do I...",
  "where is...", "what does X do", "how does X work", or any question about
  ITrack Enterprise functionality, configuration, or UI. Always use this skill
  before answering ITrack Enterprise how-to questions — do not rely on memory alone.
metadata:
  author: ISoft Data Systems, Inc.
  version: 0.1.1
  mcp-server: Enterprise-Api-Extension
  category: universal
---

# ITrack Enterprise Help Skill

## Overview

ITrack Enterprise has two distinct interfaces:

- **Web App** — modern browser-based UI
- **Desktop App** — Windows application with a different UI layout

The two apps share the same underlying data and business concepts (inventory, vehicles, customers, sales orders, permissions, etc.), but they differ in more than just UI. The desktop app communicates mostly directly with the SQL database. The web app communicates exclusively through the Enterprise API — the same API used by the Enterprise MCP connector. This means some behaviors, available fields, and workflows may differ between the two apps even for the same feature.

---

## Step 1: Classify the Question

### General / Conceptual Questions
Questions about *how the system works* — not tied to a specific UI — can be answered without knowing which app the user is on. Examples:
- "How do user permissions work?"
- "What's the difference between a sales order and a work order?"
- "What is a teardown?"
- "How does inventory status affect availability?"

→ Fetch relevant documentation from **either source** (prefer Web KB). No need to ask which app.

### UI / Workflow Questions
Questions about *where to click*, *how to navigate*, or *step-by-step workflows* are UI-specific. Examples:
- "How do I create a new vehicle?"
- "Where do I find the interchange screen?"
- "How do I add a part to a sales order?"
- "How do I configure user groups?"

→ **If the user has already stated which app they are using in this conversation, assume that for all subsequent questions without asking again.**

→ **If the user's app is unknown, ask before fetching:**
> "Are you using the ITrack Enterprise **web app** or the **desktop app**? The steps differ between them."

---

## Step 2: Route to the Right Documentation

### Web App Documentation
**Index page:** `https://itrack-enterprise.scroll.site/eewkb`
- Article URLs follow the pattern: `https://itrack-enterprise.scroll.site/eewkb/<article-slug>`

To find the right article, fetch the index page first — it lists all available articles with their URLs. Navigate to the most relevant one. If the topic isn't listed, fall back to the desktop documentation.

---

### Desktop App Documentation
**Index page:** `https://wikido.isoftdata.com/index.php?title=ITrack_Enterprise_User%27s_Guide`
- Article URLs follow the pattern: `https://wikido.isoftdata.com/index.php?title=ITrack/Enterprise/<Article_Name>`

⚠️ The wiki hosts documentation for multiple ISoft products (ITrack Pro, Chromium, Presage, etc.). **Only fetch and cite pages under the `ITrack/Enterprise/` path.** Never reference articles from other product sections — users of this skill are exclusively in the ITrack Enterprise ecosystem.

⚠️ Never fetch wiki URLs that contain any of the following query parameters — these are internal wiki editor pages, not article content: `&oldid=`, `&action=edit`, `&action=history`, `&action=info`, `&action=raw`, `&diff=`, `&redlink=`.

**Finding the right article:** The User's Guide index page is the authoritative starting point. Fetch it first and use it as the priority list of available articles.

**Handling empty pages:** The wiki has many stub pages with no content. If a fetched article is empty or returns no useful content, do not treat this as a domain block or access failure — silently skip it and try the next most relevant article from the index. Keep trying before falling back to the web KB. Only give up on the wiki after making a reasonable attempt across multiple relevant articles.

---

## Step 3: Fetch and Respond

1. Fetch the appropriate index page to locate the right article URL, then fetch the article.
2. Summarize in your own words — do not reproduce large blocks of text verbatim.
3. Cite the source URL regardless of which documentation was used.
4. If the fetched page doesn't contain the needed info, try a related article or ask the user to clarify.
5. **Surface cross-app caveats**: The documentation sometimes includes notes that a feature is only available in one app (e.g., *"Purchase Orders are only supported in ITrack Enterprise Desktop at this time."*). Always pass these through to the user — they are important context regardless of which app the user is on.
6. **If documentation does not resolve the user's question**: direct them to ISoft support as a last resort (see Coverage Gaps).

---

## Environment and Architecture Questions

For questions about how the ITrack Enterprise ecosystem is structured — hosting options, where data lives, how components connect, which apps are available, what's required vs. optional — read `references/environment.md` before responding.

This covers: SQL database hosting (cloud vs. on-prem), Enterprise API, Enterprise Web (EEW), Crystal Reports/Server, Report Queue, ITrack LX, Enterprise MCP, HTP/eCommerce stack, desktop integrations, and 3rd party development.

Use it when users ask things like:
- "Where is my data hosted?"
- "What do I need to use the web app?"
- "What's the difference between the HTP API and the Enterprise API?"
- "How does report printing work?"
- "Can I access Enterprise from my phone?"

Not every organization uses every component, and users may not be aware of components outside their own setup.

When responding to environment questions, translate technical concepts into plain language oriented around what the user can *do* or *expect*, rather than how the system works internally. Avoid surfacing internal component names (e.g. Aggregator, Report Commander, ODBC) unless the user has specifically asked about them. Focus on outcomes: where data lives, what's needed to use a feature, who to contact for setup.

---

## Version and Changelog Questions

Use changelogs when a user asks whether they're on the current version, reports a possible bug, or describes unexpected behavior — it's worth asking their version and comparing to recent releases.

**Update cadence context:**
- **Web app and API** — ISoft pushes updates to active users
- **Desktop app** — updates must be manually installed by the end user

### Changelog Sources

| Component | URL |
|---|---|
| Enterprise API | `https://changelog.isoftdata.com/?product=enterprise-api` |
| Enterprise Web | `https://changelog.isoftdata.com/?product=enterprise-web` |
| Enterprise Desktop | `https://wikido.isoftdata.com/index.php?title=ITrack/Enterprise/Changelog` |

**Note:** The `changelog.isoftdata.com` pages are JavaScript-rendered (Svelte). When fetching them with `web_fetch`, the release data is embedded as a JSON blob in the raw HTML source inside a `Promise.all` script block. Extract version entries from that JSON rather than trying to parse the rendered HTML. Look for the `releases` array in the embedded data.

The desktop wiki changelog is a plain index page linking to per-version subpages. Fetch the index to find the latest version, then fetch the relevant subpage if detail is needed.

---

## Coverage Gaps

Only surface the following if documentation has not resolved the user's question:
- **Email:** support@isoftdata.com
- **Phone:** 800-929-1829 (8:30am – 5:00pm Central), press 3 for support
- **Discord:** https://discord.gg/RZBbR4g8kd
