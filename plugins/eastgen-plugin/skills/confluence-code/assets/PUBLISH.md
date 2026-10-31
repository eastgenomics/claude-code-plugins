# Publishing to Confluence

## Use the bundled MCP server

This plugin bundles the official Atlassian remote MCP server (Confluence + Jira Cloud, v2 endpoint), authenticated per user via OAuth — no shared credentials or tokens live in this plugin. Discover its tools:

```
ToolSearch: query "+atlassian"
```

Use `createConfluenceContent` / `updateConfluenceContent` / `getConfluenceContent` with the **`"html"`** format — `content_format: "html"` plus `detail: "full"` on reads, `body: {format: "html", value}` on writes, and `cloudId: "cuhbioinformatics.atlassian.net"` on every call (see `../../shared/confluence/HTML_DIALECT.md` for the dialect and Update Mechanics; `../confluence-docs/SKILL.md` → Access lists every tool and how it's reached). They handle auth for you; updates need the `snapshotToken` from the latest fetch or update response. If a call reports the server needs authentication, that user hasn't signed in yet: direct them to run `/mcp` (interactive) or `claude mcp login atlassian` to complete their own one-time OAuth flow.

## Drafting workflow (do this before calling any tool)

1. Write each page's HTML+ body to a local scratch file first, e.g. `./confluence-drafts/{page-slug}.html`, so it stays readable and reviewable.
2. **Show the user the page plan** (titles, hierarchy, one-paragraph summary of each page) and get explicit go-ahead before creating anything. If they ask to see full drafts first, show them — these become visible to the whole space once published, so there's no "publish first, fix later" step to lean on.
3. **Create all five pages first**, even the parent — you need every page's real ID before any page can link to another (see HTML_DIALECT.md's page-links section). It's fine for the parent to start without its children list and for children to start without series navigation.
4. **Second pass:** once every ID is known, `updateConfluenceContent` the parent with its children list and each child with its series-navigation panel. Each update needs a fresh `snapshotToken` (from the create response or a re-fetch). Adding a children list or navigation panel is a natural fit for granular `edits` — `appendNodeToEnd`, or `insertNodeAfter` a known `data-local-id` — rather than resending the whole body; run it with `dryRun: true` first and read the echoed HTML.

## Confirmation gate

Before calling any page-creation or page-update tool:

1. The user has confirmed the space (and parent page, if applicable).
2. The user has seen the page plan (titles + hierarchy + one-line summary each) and said to proceed.

Creating a page is a visible, shared-state action — treat it with the same care as pushing a commit or opening a PR, not as a reversible local edit.
