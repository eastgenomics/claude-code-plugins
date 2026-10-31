---
name: confluence-docs
description: "create dev-doc | create resource-doc | create policy-doc | update — create and update controlled documentation pages in the CUH Bioinformatics Documentation Vault on Confluence."
argument-hint: "create dev-doc | create resource-doc | create policy-doc | update"
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

You are helping manage controlled documentation in the CUH Bioinformatics Documentation Vault on Confluence.

The user has invoked this skill with the following arguments:

$ARGUMENTS

---

## Mode dispatch

Read the arguments, determine the mode, then **immediately read the corresponding reference file** before doing anything else:

| Arguments start with | Reference file | Covers |
|---|---|---|
| `create dev-doc` | [references/dev-doc.md](references/dev-doc.md) | Development & Testing docs — apps, tools, config files |
| `create resource-doc` | [references/resource-doc.md](references/resource-doc.md) | Controlled File docs — data files, variant stores, reference resources |
| `create policy-doc` | [references/policy-doc.md](references/policy-doc.md) | Manual/Policy docs — operational guidance, training policies |
| `update` | [references/update.md](references/update.md) | Surgical edits to any existing page |

Text after the mode keyword is additional context (title, URLs, description of the change) — carry it through into the reference file's workflow. If no valid mode is given, ask which of the four is needed.

The three `create` modes all clone their page from a live template page (see **Live template pages** below) rather than building HTML+ from scratch — read that section before starting any `create` workflow.

Read [../../shared/confluence/HTML_DIALECT.md](../../shared/confluence/HTML_DIALECT.md) once per session before writing any page body — every mode below builds HTML+ content (panels, status lozenges, expand sections, Mermaid), and that reference is the single source of truth for the exact shapes.

---

## Org constants

- **Base URL**: `https://cuhbioinformatics.atlassian.net/wiki`
- **Space key**: `DV`
- **Space ID**: `2903080965`
- **Cloud ID**: use the site URL `cuhbioinformatics.atlassian.net` directly as `cloudId` in every MCP call below — the tools resolve it automatically. (The underlying UUID `3419b8d5-6218-492f-bff8-812d5d24cdc7` is only needed for the Mermaid macro's `embeddedMacroContext` — see HTML_DIALECT.md → Mermaid.)

Confirm these are still current with `executeRead({name: "getConfluenceSpace", cloudId: "cuhbioinformatics.atlassian.net", inputs: {spaceIdOrKey: "DV"}})` if anything looks off — they're recorded here to skip a lookup on every run, not treated as unchangeable.

## Access

Use the bundled `atlassian` MCP server — the **v2 endpoint** (`https://mcp.atlassian.com/v2/mcp`), OAuth-authenticated per user, no token file. v1 lacks the attachment tools and uses different tool names. Find its tools with `ToolSearch: query "+atlassian"` (not `"confluence"` — the discovery-layer tools don't have "confluence" in their names). If a call reports the server needs authentication, direct the user to `/mcp` or `claude mcp login atlassian`.

v2 exposes a few **primary** tools directly and the rest through a discovery layer (split confirmed in Claude Code on 2026-10-31): `discover(query)` finds an operation, then it is run with `executeRead` / `executeWrite` / `executeDestructive` (`name`, `cloudId`, `inputs`). Call primary tools directly. In Claude Code every tool is prefixed `mcp__plugin_eastgen-plugin_atlassian__`.

Tools used by the Confluence skills:

| Tool | Kind | Purpose |
|---|---|---|
| `getConfluenceContent` | primary | Fetch a page. Use `content_format: "html"` **and `detail: "full"`** (default `detail` is a cheap summary with no body). Returns `snapshotToken` and `data.body.value`. Accepts `content_id` or `content_url`. |
| `createConfluenceContent` | primary | Create a page: `parent: {spaceId, parentContentId}`, `contentType: "page"`, `title`, `body: {format:"html", value:"…"}` |
| `updateConfluenceContent` | primary | Update a page: either `body: {format:"html", value}` (full replace) **or** `edits: [...]` (granular), plus the `snapshotToken` from the latest fetch. `dryRun: true` validates without saving. |
| `searchConfluence` | primary | Find pages by title/text/label (CQL) |
| `atlassianUserInfo` | primary | Current user's `accountId` (e.g. for the Mermaid macro) |
| `getConfluenceContentDescendants` (`contentId`, `depth`) | `executeRead` | List child pages |
| `getConfluenceSpace` (`spaceIdOrKey`) | `executeRead` | Space ID/key/homepage + space instructions |
| `createConfluenceAttachment` | `executeWrite` | Prepare a file upload (returns a curl command) — see HTML_DIALECT.md → Images and Attachments |
| `listConfluenceAttachments` / `getConfluenceAttachment` / `downloadConfluenceAttachment` | `executeRead` | Inspect / download attachments |
| `getContentFormatGuide` (`toolName: "createConfluencePage"` / `"updateConfluencePage"`) | `executeRead` | Canonical HTML format reference — call before authoring unfamiliar markup |
| `getConfluencePersonalSpace` | `executeRead` | The current user's personal space (ID and `~` key) — use it for throwaway test pages, never DV |

Parameter casing is mixed, and that is the real schema: `getConfluenceContent` takes snake_case (`content_id`, `content_format`), while `createConfluenceContent` / `updateConfluenceContent` take camelCase (`contentId`, `contentType`, `snapshotToken`) — copy names exactly, don't normalise them.

Always pass `cloudId` explicitly (the site URL `cuhbioinformatics.atlassian.net` works). Space instructions: before authoring in a space, load them with `getConfluenceSpace` (via `executeRead`) and apply them to the whole body — **unless** a `getConfluenceContent` read in that same space already returned `metadata.hasSpaceInstructions: false`. Treat `true`, a missing flag, or no prior read (e.g. creating a page in a space you haven't read from) as "load them". Load them at most once per space per task. (`DV` returned no instructions when last checked.)

There is no MCP tool for fetching a Confluence page **template** (the admin-defined kind). That's fine — the org no longer uses that feature for these doc types. Instead, each doc type has a **live Confluence page that is functionally the master template**: fetch it fresh with `getConfluenceContent` (`content_format: "html"`, `detail: "full"`), clone its body, and replace placeholders per that reference file's Content Guide. See **Live template pages** below.

## Live template pages

Under [Live documentation guide](https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/3305504903/Live+documentation+guide):

| Doc type | Template page ID | URL |
|---|---|---|
| Development & Testing | `4745560161` | [.../pages/4745560161](https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/4745560161/Development+Testing+doc+template+copy+this+page+s+content+to+create+a+new+dev-doc) |
| Controlled File | `4745363735` | [.../pages/4745363735](https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/4745363735/Controlled+File+doc+template+copy+this+page+s+content+to+create+a+new+resource-doc) |
| Manual / Policy | `4745363752` | [.../pages/4745363752](https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/4745363752/Manual+Policy+doc+template+copy+this+page+s+content+to+create+a+new+policy-doc) |

These three pages are literal copies of the org's former Confluence templates — they exist purely to be fetched and cloned, never to be linked to a user as a finished document.

Fetching one with `content_format: "html"` returns its body already translated into this dialect's macro shapes: Page Properties panel (`bodied-extension`), TOC (`extension`), `expand` (plain `<details><summary>`), `tip` → `<div data-type="panel-success">`, status lozenges (`<span data-type="status" data-color="...">`), and placeholders (`<span data-type="placeholder">prompt text</span>`) — verified live on v2 (`getConfluenceContent`, `detail: "full"`): all three template pages return exactly these shapes. All three templates now use a Page Properties `bodied-extension` (`details`) around the header table — the Manual/Policy template was a plain table until it was wrapped on 2026-10-31 (page version 2); clones taken before then lack the macro (see `references/update.md` → Wrapping an old plain header table).

**Clone-and-create procedure — single source of truth for all three `create` modes:**

1. `getConfluenceContent(cloudId, content_id: "<template page ID from the table above>", content_format: "html", detail: "full")` — returns the full body string in `data.body.value`.
2. Walk that body and replace each `<span data-type="placeholder">...</span>` (and the default Page Properties values) with real content per that mode's Content Guide. Leave a placeholder untouched, verbatim, for anything not yet known — don't invent content.
3. Create the page in one call with the edited body:
   ```
   createConfluenceContent(cloudId, parent: {spaceId, parentContentId}, contentType: "page", title, body: {format: "html", value: <html>})
   ```
4. Report the returned page's URL.

**Full-size clones verified on v2** (all three templates cloned with `data-local-id`s stripped, into a personal space): the outline (headings) matched the source exactly, and element counts matched — Dev doc: 5 expands, 9 status lozenges, 6 success panels, 13 emoji, 6 code blocks; Controlled File: 4 expands, 8 lozenges, 5 panels; Policy: panel-note, TOC, headings. Differences: the server adds `_parentId` to macro params, and zero-width spaces in the source can be dropped. A ~25 KB body creates in one call.

**Cloned bodies may keep or drop `data-local-id`s** — both were tested and both create fine (the server regenerates IDs). Keep `macroId` and `data-parameters` inside macros verbatim. After create, the server adds `_parentId` to macro params on the stored copy; ignore it.

Don't reconstruct the skeleton from memory of the Content Guide — always start from the cloned template body, which already has every macro (Page Properties, TOC, `expand`, `tip`→`panel-success`, status lozenges, and — for the dev-doc template — repeated Test Scenario blocks) in the exact shape Confluence expects.

**If the org's documentation conventions change**, update these three pages directly in Confluence (UI edit, or re-copy from an updated source) — nothing in this skill needs to change as long as each page's section names still match the Content Guide in its reference file.

---

## Shared conventions across all modes

**Resolving a parent page ID from a URL:** `https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/1234567890/...` → `1234567890`.

**Status lozenges** (see HTML_DIALECT.md): DRAFT = yellow, APPROVED = green, FAIL = red.

**Heading emojis:** each *distinct* H1 section gets a unique animal emoji (Summary 🐮, Overall Outcome 🦉, Useful linked documentation 🐱, Design 🐼, Changes 🦧, PRs 🐦, Test setup 🐸, Testing 🐧, Controlled file summary 🐰, Provenance 🦅, Purpose 🐸, Process 🐠 …) — confirmed against the live template pages. **Repeated/templated subsections reuse the same emoji on every repetition** — e.g. every `Test Scenario Title` H2 in the dev-doc template repeats 🐕‍🦺, and every one in the resource-doc template repeats 🦮. Don't invent a fresh emoji per repeated instance. Since `create` modes clone from the live template pages, this is inherited automatically — only assign a new emoji by hand when adding a genuinely new top-level section that doesn't already exist in the cloned skeleton.

**Done when** (create modes): the page exists in Confluence with every Content Guide section populated (or explicitly marked "To be completed" where the reference allows it), and its URL has been reported back.

**Done when** (update mode): the specific requested change is applied, verified by re-reading the page, and any remaining manual step (permissions, archiving, ticket updates) from that reference file's Post-Update Reminders has been surfaced to the user.
