# Confluence HTML+ dialect (via the bundled `atlassian` MCP server)

Shared reference for every skill in this plugin that reads or writes Confluence pages. Single source of truth — skills point here rather than restating this.

The bundled `atlassian` MCP server is the **v2 endpoint** (`https://mcp.atlassian.com/v2/mcp`). Its Confluence tools (`getConfluenceContent`, `createConfluenceContent`, `updateConfluenceContent`, …) take a body format of `"html"` or `"markdown"` (`content_format` on reads, `body: {format, value}` on writes). **Use `"html"`** — it round-trips cleanly (confirmed live against this vault) and is what the tool descriptions document in full. Reads must also pass **`detail: "full"`** — the default `detail` is a cheap summary with no body. These tools **reject Confluence storage-format XML** (`<ac:structured-macro>`) — never use it, even though it's what the raw Confluence REST API and its templates use internally.

For the canonical format reference, `getContentFormatGuide` (`toolName: "createConfluencePage"` or `"updateConfluencePage"` — the server's internal operation names, not the `…Content` tool names) — call it before authoring unfamiliar markup. How each tool is reached (directly or through the server's discovery layer) is listed once, in `../../skills/confluence-docs/SKILL.md` → Access.

## Basic blocks

Standard HTML: `<h1>`–`<h6>`, `<p>`, `<table>`/`<thead>`/`<tbody>`/`<tr>`/`<th>`/`<td>`, `<a href="URL">`, `<pre><code class="language-python">`, `<ul>`/`<ol>`/`<li>`. Never wrap the body in `<html>`, `<head>`, or `<body>`.

## Confluence-specific nodes (`data-type` attributes, not CSS classes)

| Purpose | Shape |
|---|---|
| Panel | `<div data-type="panel-info\|panel-warning\|panel-note\|panel-success\|panel-error"><p>text</p></div>` |
| Status lozenge | `<span data-type="status" data-color="green\|red\|yellow\|blue\|neutral\|purple">Label</span>` |
| Expand (collapsible) | `<details><summary>Title</summary><p>content</p></details>` |
| Placeholder | `<span data-type="placeholder">prompt text</span>` |
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

Don't rely on native `toc` or `children`-display macros for auto-built navigation of *sibling links* — they display, not link cross-page. Hand-build a short links list in the second pass instead; you already have the real IDs by then, and it's one paragraph.

## Update Mechanics

v2 supports two safe write styles; both start from a fresh `getConfluenceContent(content_format="html", detail="full")`, which yields the body and `snapshotToken`.

- **Full-body replace** (default for the controlled-doc workflows): change only the target span of HTML, keeping everything else byte-for-byte, then `updateConfluenceContent(body={format:"html", value}, snapshotToken)`.
- **Granular `edits`** (good for a single cell/paragraph/append): `replaceNode` / `insertNodeAfter` / `appendNodeToEnd` etc., targeting a node by its fetched `data-local-id`. Use `dryRun: true` first to preview the resulting HTML. The server can still re-serialise markup outside the targeted node (nesting order, entity forms, regenerated `data-local-id`s; occasionally a lost trailing space or table `data-layout` — see `../../skills/confluence-docs/references/update.md` → Wrapping an old plain header table → Lessons), so compare the whole dry-run output, and the re-fetched body after the real call, against the original before accepting it.

Either way: two or three tool calls (fetch → [dry run] → update), then re-fetch to verify. See `../../skills/confluence-docs/references/update.md` for the full workflow, including sign-off, section-replacement, and table-row patterns.

## Mermaid flowcharts — verified, this org's real mechanism

Confirmed live against `cuhbioinformatics.atlassian.net` (space `DV`), including an actual write: created a test page, added this exact shape via an update, re-fetched, and the extension node survived intact. The vault renders Mermaid via a **Forge macro** ("Mermaid diagram" app). Each diagram is two elements: a code block holding the Mermaid source, immediately followed by the extension `<div>`. The macro reads the **Nth code block on the page, in document order** (0-based `guestParams.index`) — **counting every `<pre><code>` block on the page, not just ones intended for Mermaid.** A code block that has nothing to do with Mermaid still consumes an index slot if it appears earlier in the document. This was confirmed the hard way: a bash example block placed before a Mermaid diagram, with the diagram's `index` set to `0`, rendered the diagram widget pointed at the bash block instead — the fix was setting `index` to `1` (its true position counting all code blocks, not just Mermaid ones).

```html
<details data-breakout="wide"><summary>mermaid</summary><pre><code>flowchart TD
    A[Parse config] --> B{Valid?}
    B -->|yes| C[Run pipeline]
    B -->|no| D[Raise error]
</code></pre></details><div data-type="extension" data-extension-key="23392b90-4271-4239-98ca-a3e96c663cbb/63d4d207-ac2f-4273-865c-0240d37f044a/static/mermaid-diagram" data-extension-type="com.atlassian.ecosystem" data-layout="default" data-parameters="{&quot;layout&quot;:&quot;extension&quot;,&quot;guestParams&quot;:{&quot;index&quot;:0},&quot;forgeEnvironment&quot;:&quot;PRODUCTION&quot;,&quot;embeddedMacroContext&quot;:{&quot;accountId&quot;:&quot;{ACCOUNT_ID}&quot;,&quot;cloudId&quot;:&quot;3419b8d5-6218-492f-bff8-812d5d24cdc7&quot;,&quot;contextIds&quot;:[&quot;ari:cloud:confluence:3419b8d5-6218-492f-bff8-812d5d24cdc7:workspace/b350d1a9-a5de-4ffd-92ef-1992dfdf8f23&quot;],&quot;extensionData&quot;:{&quot;type&quot;:&quot;macro&quot;,&quot;content&quot;:{&quot;id&quot;:&quot;{PAGE_ID}&quot;,&quot;type&quot;:&quot;page&quot;,&quot;version&quot;:1},&quot;space&quot;:{&quot;id&quot;:&quot;{SPACE_ID}&quot;,&quot;key&quot;:&quot;{SPACE_KEY}&quot;}}},&quot;localId&quot;:&quot;{LOCAL_ID}&quot;,&quot;extensionId&quot;:&quot;ari:cloud:ecosystem::extension/23392b90-4271-4239-98ca-a3e96c663cbb/63d4d207-ac2f-4273-865c-0240d37f044a/static/mermaid-diagram&quot;,&quot;extensionTitle&quot;:&quot;Mermaid diagram&quot;}"></div>
```

**Fill in per use:**
- `{ACCOUNT_ID}` — the current user's account ID (`atlassianUserInfo` MCP tool, or read from an existing page you fetched).
- `{PAGE_ID}` — **the page must already exist first** — this is a chicken-and-egg constraint like the page-links one above: create the page (with placeholder content or without its diagrams), then `updateConfluenceContent` to add Mermaid diagrams once you know the page's own ID.
- `{SPACE_ID}` / `{SPACE_KEY}` — from `getConfluenceSpace` (via `executeRead`) (`2903080965` / `DV` for this vault).
- `{LOCAL_ID}` — any unique string per diagram (e.g. a short slug), used only for the inner `localId` in `data-parameters`. Don't add an outer `data-local-id` to the new `<div>` (see the rule on new nodes above).
- `index` — 0-based position counting **every** code block on the page in document order, Mermaid or not (see gotcha above) — not just the Mermaid ones.

Cloud ID (`3419b8d5-6218-492f-bff8-812d5d24cdc7`) and the workspace context ARI are site-wide constants for `cuhbioinformatics` — stable across pages and spaces on this site, safe to hardcode.

**Gotchas:**
- Confluence silently drops macros the site can't resolve — after writing, re-fetch the page (`content_format: "html"`, `detail: "full"`) and confirm the extension `<div>` with `mermaid-diagram` in `data-extension-key` survived.
- Keep Mermaid labels single-line (`<br/>` instead of literal newlines) — `subgraph`, `flowchart`, and edge labels (`-->|"text"|`) all render fine.

**Verified on v2** (test page in a personal space): appending the `<details>` code block and the extension `<div>` as two `appendNodeToEnd` edits worked; on re-fetch the extension node survived with `guestParams.index` intact and the Mermaid source intact. The `index` must count every other code block on the page (the test page had one earlier `<pre><code>`, so the Mermaid block was index 1). The macro's rendering was not checked visually, only that the node persisted. For `{SPACE_ID}`/`{SPACE_KEY}` in a personal space use that space's own ID and `~accountNumber` key.

## Verified live (this vault, `cuhbioinformatics.atlassian.net`)

- `createConfluenceContent` / `updateConfluenceContent` with `format: "html"` translate panels, status lozenges, expand sections, tables, and code blocks to native Confluence macros (originally confirmed on v1 by round-tripping the three template pages; v2 round-trips media, see below).
- v2 `updateConfluenceContent` needs a **`snapshotToken`** (from the latest `getConfluenceContent`/`updateConfluenceContent` response — `v:<n>`) rather than a version number. A stale token fails with `snapshot_stale`: re-fetch, reapply, retry. It does not need the version number to be incremented by hand.
- v2 **does** have partial updates: `edits: [{name, localId, value, arguments}]` with `replaceNode`, `insertNodeAfter`, `insertNodeBefore`, `deleteNode`, `removeNodes`, `moveNode`, `appendNodeToEnd`, `setAttrs`. `localId` is the `data-local-id` of a fetched node (not needed for `appendNodeToEnd`). `body` and `edits` are mutually exclusive. `replaceNode` assigns the new node a fresh `data-local-id`. Verified live: `appendNodeToEnd`, `replaceNode` (several per call, including inside a Page Properties table, with a status lozenge and a `<time>` date), `dryRun`.
- **`replaceNode` must target exactly the node type you replace.** `localId` on a `<td>` with a `<p>` value **removes the cell and hoists the paragraph out of the table** (observed live; corrupted a Page Properties table). To change a cell's text, target the **`<p>` inside it** and send a `<p>`; to replace a whole cell, target the `<td>` and send `<td><p>…</p></td>`. In a fetched body, the `td`'s `data-local-id` comes first and the `p`'s second — don't mix them up. Always `dryRun: true` a multi-edit batch and read the echoed HTML first.
- Re-verified in Claude Code (2026-10-31, private throwaway page in a personal space): a published-version fetch's `snapshotToken` works for `updateConfluenceContent` — the tool description suggests fetching with `draft: true`, but that isn't needed for an ordinary publish. `createConfluenceContent` also returns `snapshot.token`, usable for the page's first update without a re-fetch. A `dryRun` batch of two `replaceNode` (on the `<p>` inside table cells) + one `appendNodeToEnd` returned `dry_run_validated` with the table intact, and the page version stayed at 1.
- `setAttrs` passed dry-run validation but had **no visible effect** when tried on `data-background` — don't rely on it; use `replaceNode` instead.
- `<code class="language-bash">` came back as `language-shell`; Mermaid source in a `<pre><code>` round-trips (`-->` is escaped as `--&gt;` in the HTML, which is fine).
- Confluence silently drops macros the site can't resolve — after writing, re-fetch the page (`content_format: "html"`, `detail: "full"`) and confirm anything unusual (e.g. a Forge macro extension node) survived.

## Images and Attachments

Verified live on v2 (image attached to and embedded in a page). Local files **cannot** be referenced by path in a body, and the MCP server never receives file bytes — upload is a **prepare (MCP) → upload (local curl) → embed (MCP)** sequence:

1. **Prepare**: `createConfluenceAttachment` (via `executeWrite`) with `cloudId` and `inputs={contentId, localFilePath, fileSize}`. `fileSize` is optional but **must be ≥ 1 if passed** (get it with `stat -c %s`); don't pass `0`. The page must already exist. Returns `data.uploadCommand`, a ready-made `curl … --form 'file=@"<path>"'` with a short-lived bearer token (max 100 MB).
2. **Upload**: run `uploadCommand` verbatim with Bash. The JSON response contains `id` (`att…`), `title`, `mediaType`, and **`fileId`** (a UUID). The `fileId` is the media ID. Don't echo the bearer token back to the user or paste it in files.
3. **Embed**: the collection is `contentId-<pageId>`. Add the HTML below with `updateConfluenceContent` — easiest as `edits: [{"name":"appendNodeToEnd","arguments":{},"value":"<html>"}]` with the current `snapshotToken` (or `insertNodeAfter` a known `localId` for positional placement), or include it in a full `body`.

| Display | HTML |
|---|---|
| Image (block) | `<figure data-type="media-single" data-layout="center" data-width="80" data-width-type="percentage"><div data-type="media" data-media-type="file" data-id="FILE_ID" data-collection="contentId-PAGE_ID" data-alt="name.png"></div></figure>` |
| File card(s) | `<div data-type="media-group"><div data-type="media" data-media-type="file" data-id="FILE_ID" data-collection="contentId-PAGE_ID" data-alt="file.pdf"></div></div>` |
| Inline file/image | `<p>See <span data-type="media-inline" data-media-type="file" data-media-id="FILE_ID" data-media-collection="contentId-PAGE_ID" data-alt="file.pdf"></span>.</p>` |

Notes:
- Add `<figcaption>…</figcaption>` inside the `<figure>` for a caption. Prefer percentage widths (80 diagrams/charts, 60 screenshots).
- Confluence may rewrite a requested percentage width to a pixel width (observed: `80%` → `544px` for a 3780×1736 PNG). Re-fetch to check, and re-edit with `replaceNode` if it renders too small.
- External images need no upload: `<img src="https://…" alt="…" width="400">` (public URL only).
- `listConfluenceAttachments` (`contentId`) shows what is already attached; use its `fileId`/`id` rather than guessing. Never invent `data-id` / `data-collection` values.
- Download: `downloadConfluenceAttachment` (`attachmentId`, `outputPath`) returns a local curl command, same pattern.
- Registered documents in this vault are controlled — only attach files to a controlled page when the user asks for it.
