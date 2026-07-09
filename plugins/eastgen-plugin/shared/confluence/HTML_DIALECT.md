# Confluence HTML+ dialect (via the bundled `atlassian` MCP server)

Shared reference for every skill in this plugin that reads or writes Confluence pages. Single source of truth — skills point here rather than restating this.

The bundled `atlassian` MCP tools (`createConfluencePage`, `updateConfluencePage`, `getConfluencePage`, …) take a `contentFormat` of `"html"`, `"markdown"`, or `"adf"`. **Use `"html"`** — it's the format the tools document in full detail and round-trips cleanly, including through edits. The tools explicitly reject Confluence storage-format XML (`<ac:structured-macro>`) — don't use it, even though it's what raw Confluence REST API examples elsewhere use.

## Basic blocks

Standard HTML: `<h1>`–`<h6>`, `<p>`, `<table>`/`<thead>`/`<tbody>`/`<tr>`/`<th>`/`<td>`, `<a href="URL">`, `<pre><code class="language-python">`, `<ul>`/`<ol>`/`<li>`. Never wrap the body in `<html>`, `<head>`, or `<body>`.

## Confluence-specific nodes (`data-type` attributes, not CSS classes)

| Purpose | Shape |
|---|---|
| Panel | `<div data-type="panel-info\|panel-warning\|panel-note\|panel-success\|panel-error"><p>text</p></div>` |
| Status lozenge | `<span data-type="status" data-color="green\|red\|yellow\|blue\|neutral\|purple">Label</span>` |
| Expand (collapsible) | `<details><summary>Title</summary><p>content</p></details>` |
| Task list | `<ul data-type="task-list"><li data-type="task-item"><input type="checkbox"> Item</li></ul>` |
| Decision list | `<ul data-type="decision-list"><li data-type="decision-item" data-state="DECIDED\|UNDECIDED">text</li></ul>` |
| Layout (columns) | `<section data-type="layout-two-equal\|layout-three-equal"><div data-type="column"><p>Column</p></div></section>` |
| Inline card / smart link | `<a href="URL" data-card-appearance="inline">text</a>` |
| Date | `<time datetime="YYYY-MM-DD">label</time>` |
| Mention | `<span data-type="mention" data-user-id="ACCOUNT_ID">@Name</span>` |
| Generic macro/extension | `<div data-type="extension\|bodied-extension\|multi-bodied-extension" data-extension-key="KEY" data-extension-type="TYPE" data-parameters="JSON"></div>` |

For **native Confluence macros**, `data-extension-type="com.atlassian.confluence.macro.core"`. Omit `macroId`/`data-local-id` on new nodes — only copy `data-local-id`, `data-user-id`, `data-id`, `data-collection`, `data-media-id`, `data-resource-id` from content you fetched, never invent them.

**Nesting rules:** task/decision items, headings, and captions are inline-only. List items can't contain headings/tables/panels/expands. Panels can't contain tables/expands/blockquotes/panels. Table cells can contain headings, panels, lists, and a nested expand (`<details data-type="nested-expand">`) but not nested tables or a normal expand.

## Tables

Support `data-number-column`, `data-layout="default|center|wide|full-width"`, `data-width`, row/col spans, `data-colwidth`, and cell `data-background`. Breakout (wide layout) is available on `details`, `pre`, `section`, and sync blocks via `data-breakout="wide|full-width"`.

## Page links — no lazy title-based linking

Unlike storage-format XML, this dialect has no `<ri:page ri:content-title="...">`-style link that resolves by title at render time. Every link is a plain `<a href="...">` to a real URL, which means **you need the target page's real ID before you can link to it.**

**Implication for any multi-page set (a series, a parent + children):** create every page first, collecting each returned page ID, then go back and update pages that need to link to siblings — a two-pass process, not one. Page URLs take the form `{baseUrl}/wiki/spaces/{spaceKey}/pages/{pageId}/{url-encoded-title}` (the title-encoded suffix is optional; Confluence redirects correctly from just `{baseUrl}/wiki/spaces/{spaceKey}/pages/{pageId}`).

Don't rely on native `toc` or `children`-display macros for auto-built navigation — their `data-extension-key` values aren't verified for this dialect. Hand-build a short links list in the second pass instead; you already have the real IDs by then, and it's one paragraph.

## Mermaid flowcharts — verified, this org's real mechanism

Confirmed live against `cuhbioinformatics.atlassian.net` (space `DV`), including an actual write: created a test page, added this exact shape via `updateConfluencePage`, re-fetched, and the extension node survived intact. The vault renders Mermaid via a **Forge macro** ("Mermaid diagram" app). Each diagram is two elements: a code block holding the Mermaid source, immediately followed by the extension `<div>`. The macro reads the **Nth code block on the page, in document order** (0-based `guestParams.index`) — **counting every `<pre><code>` block on the page, not just ones intended for Mermaid.** A code block that has nothing to do with Mermaid still consumes an index slot if it appears earlier in the document. This was confirmed the hard way: a bash example block placed before a Mermaid diagram, with the diagram's `index` set to `0`, rendered the diagram widget pointed at the bash block instead — the fix was setting `index` to `1` (its true position counting all code blocks, not just Mermaid ones).

```html
<details data-breakout="wide"><summary>mermaid</summary><pre><code>flowchart TD
    A[Parse config] --> B{Valid?}
    B -->|yes| C[Run pipeline]
    B -->|no| D[Raise error]
</code></pre></details><div data-type="extension" data-extension-key="23392b90-4271-4239-98ca-a3e96c663cbb/63d4d207-ac2f-4273-865c-0240d37f044a/static/mermaid-diagram" data-extension-type="com.atlassian.ecosystem" data-layout="default" data-parameters="{&quot;layout&quot;:&quot;extension&quot;,&quot;guestParams&quot;:{&quot;index&quot;:0},&quot;forgeEnvironment&quot;:&quot;PRODUCTION&quot;,&quot;embeddedMacroContext&quot;:{&quot;accountId&quot;:&quot;{ACCOUNT_ID}&quot;,&quot;cloudId&quot;:&quot;3419b8d5-6218-492f-bff8-812d5d24cdc7&quot;,&quot;contextIds&quot;:[&quot;ari:cloud:confluence:3419b8d5-6218-492f-bff8-812d5d24cdc7:workspace/b350d1a9-a5de-4ffd-92ef-1992dfdf8f23&quot;],&quot;extensionData&quot;:{&quot;type&quot;:&quot;macro&quot;,&quot;content&quot;:{&quot;id&quot;:&quot;{PAGE_ID}&quot;,&quot;type&quot;:&quot;page&quot;,&quot;version&quot;:1},&quot;space&quot;:{&quot;id&quot;:&quot;{SPACE_ID}&quot;,&quot;key&quot;:&quot;{SPACE_KEY}&quot;}}},&quot;localId&quot;:&quot;{LOCAL_ID}&quot;,&quot;extensionId&quot;:&quot;ari:cloud:ecosystem::extension/23392b90-4271-4239-98ca-a3e96c663cbb/63d4d207-ac2f-4273-865c-0240d37f044a/static/mermaid-diagram&quot;,&quot;extensionTitle&quot;:&quot;Mermaid diagram&quot;}"></div>
```

**Fill in per use:**
- `{ACCOUNT_ID}` — the current user's account ID (`atlassianUserInfo` MCP tool, or read from an existing page you fetched).
- `{PAGE_ID}` — **the page must already exist first** — this is a chicken-and-egg constraint like the page-links one above: create the page (with placeholder content or without its diagrams), then `updateConfluencePage` to add Mermaid diagrams once you know the page's own ID.
- `{SPACE_ID}` / `{SPACE_KEY}` — from `getConfluenceSpaces`.
- `{LOCAL_ID}` — any unique string per diagram (e.g. a short slug); reuse the same value for both the outer `data-local-id` and the inner `parameters.localId`/`extensionId` local id if adding one.
- `index` — 0-based position counting **every** code block on the page in document order, Mermaid or not (see gotcha above) — not just the Mermaid ones.

Cloud ID (`3419b8d5-6218-492f-bff8-812d5d24cdc7`) and the workspace context ARI are site-wide constants for `cuhbioinformatics` — stable across pages and spaces on this site, safe to hardcode.

**Gotchas:**
- Confluence silently drops macros the site can't resolve — after writing, re-fetch the page (`contentFormat: "html"`) and confirm the extension `<div>` with `mermaid-diagram` in `data-extension-key` survived.
- Keep Mermaid labels single-line (`<br/>` instead of literal newlines) — `subgraph`, `flowchart`, and edge labels (`-->|"text"|`) all render fine.

## Verified live (2026-07-09, throwaway page in a personal space)

- `createConfluencePage` and `updateConfluencePage` with `contentFormat: "html"` correctly translate panels, status lozenges, expand sections, tables, and code blocks to native Confluence macros — confirmed by re-fetching and inspecting the round-tripped body.
- `updateConfluencePage` needs **no version number** — it auto-increments internally. Don't fetch-then-increment; just call it.
- `updateConfluencePage` is a full-body replace, as expected.
- The Mermaid mechanism above survives a real create → update → re-fetch cycle, and — after correcting the `index` rule above based on an actual rendering failure — now visually renders the right diagram, confirmed by the person who built this test page.
- **Gotcha:** a `<code class="language-bash">` block round-trips as `language-shell` — Confluence normalizes the language name. Don't assume the language string you send is the one you'll see back.

Not yet tested: page-to-page linking with real IDs (the two-pass pattern), and editing an existing page's specific span without touching the rest (described in `update.md` but not exercised end to end). Verify these against a throwaway page too before relying on them for something that matters.
