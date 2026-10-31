# Update — Editing Existing Pages

Use this reference when the mode is `update`. Follow these steps in order.

Every edit starts from a fresh fetch of the page's HTML+ body and its `snapshotToken`. Then either send back a **full-body replace** (change only the target span, keep everything else byte-for-byte) or a batch of granular **`edits`** that target nodes by their fetched `data-local-id`. See [../../../shared/confluence/HTML_DIALECT.md](../../../shared/confluence/HTML_DIALECT.md) → Update Mechanics.

---

## Step 1: Identify the Target Page

Ask for anything not provided in the arguments:

- **Page URL or page ID**: the page to be edited
- **What needs changing**: a clear description of the edit — e.g. "update Status to APPROVED and add Approved by Katherine Winfield", "add a new test scenario under Testing", "correct the Jira ticket link"

Extract the page ID from the URL if given: `https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/1234567890/...` → `1234567890`.

If only a title or description is given, use `searchConfluence` to find it — by title if you have one, otherwise by page text:
```
searchConfluence(cloudId: "cuhbioinformatics.atlassian.net", cql: "space = DV AND title ~ \"<title text>\" AND type = page")
searchConfluence(cloudId: "cuhbioinformatics.atlassian.net", cql: "space = DV AND text ~ \"<description keywords>\" AND type = page")
```
Show the user the candidate page(s) (title + URL) and get confirmation of which one to edit before fetching or changing anything.

---

## Step 2: Inspect the Body First

**Always fetch and read the body before writing any edit.** You cannot safely target text without knowing the page's actual current content.

```
getConfluenceContent(cloudId: "cuhbioinformatics.atlassian.net", content_id: "<page_id>", content_format: "html", detail: "full")
```

Read the returned body (`data.body.value`) to:
- Confirm the exact text and surrounding markup of what you intend to change
- Check whether a given text value appears more than once (e.g. "N/A" may appear in many cells — include enough surrounding context to make the target unique, the same discipline as any exact-match text edit)
- Identify table structure (how many rows, what labels are in the first column) before editing a specific cell
- Note any `data-local-id` / `data-color` / other attributes on the node you're changing — preserve everything you're not explicitly told to change, verbatim

If the fetch didn't return `metadata.hasSpaceInstructions: false`, load the space instructions per `../SKILL.md` → Access before editing.

Only write the edit once you've seen the actual body.

---

## Step 3: Make the Edit and Send It Back

Change only the specific span of HTML that needs to change — keep everything else byte-for-byte as fetched — then call:

```
updateConfluenceContent(cloudId: "cuhbioinformatics.atlassian.net", contentId: "<page_id>", snapshotToken: "<token from the fetch>", title: "<title>", body: {format: "html", value: "<edited html>"})
```

`updateConfluenceContent` needs the **`snapshotToken`** from the fetch (not a version number); if it returns `snapshot_stale`, re-fetch and reapply. For a small change you may instead use `edits` (e.g. `replaceNode` on the target's `data-local-id`) — see HTML_DIALECT.md → Update Mechanics. Pass the existing title unchanged unless the title itself is part of the edit.

**Safety rules:**
- Include enough surrounding markup that the targeted text is uniquely identified before changing it
- Change only the specific nodes that need to change, keeping everything else — including `data-local-id` attributes — verbatim
- Re-fetch after the update and confirm the change landed as intended — compare the whole body with the original, not just the edited span (the server can re-serialise other markup; see HTML_DIALECT.md → Update Mechanics)

---

## Common Update Scenarios

### Sign-off: update the controlled document header

Applies to all document types. Updates the status lozenge, the Approved by cell, and — only where the header has one — the Approved and locked date cell. Only the Manual / Policy header has an "Approved and locked date" row (see Document-Type Field Reference below); Development & Testing and Controlled File headers don't, so for those apply changes 1 and 2 only and don't invent a date row.

Fetch the body, then apply the applicable changes before sending it back once:

1. Status lozenge → change only `data-color="yellow"` → `"green"` and the text `DRAFT` → `APPROVED`; keep every other attribute on the `<span>` (e.g. `data-status-style="bold"`, `data-local-id`) verbatim
2. "Approved by" table cell → find the `<tr>` whose first `<td>` is `<p>Approved by</p>`, replace the second `<td>`'s contents (or the `<span data-type="placeholder">...</span>` inside it) with the approver's name
3. "Approved and locked date" table cell (Manual / Policy only) → same pattern, replace with the date (`YYYY-MM-DD`)

The same sign-off can be done as one batch of granular edits — see **Granular edits for sign-off** below.

### Update the Jira ticket link

Find the `<a href="https://cuhbioinformatics.atlassian.net/browse/DI-XXXX">DI-XXXX</a>` and replace both the visible text and `href` with the new ticket.

### Rewrite the content under a heading

Replace everything between the target `<h1>`/`<h2>` and the next heading of the same or higher level (e.g. rewriting a test scenario narrative, updating the Changes section for a new version).

### Add a new test scenario section

Insert a new `<h2>` block plus its content immediately before the next `<h1>` (or at the end of the document if there isn't one) — don't touch the existing scenarios.

### Update the page title (e.g. remove DRAFT)

Pass the new title in the same `updateConfluenceContent` call as any body change — no need for a separate call.

### Update a single table cell by label

Find the `<tr>` whose first `<td>`/`<th>` matches the label, then replace the contents of its second cell.

### Add a row to a table

Insert a new `<tr>` immediately before the table's closing `</tbody></table>`.

---

## Granular edits for sign-off (verified on v2)

Instead of resending the whole body, a sign-off can be done as one batch of `edits` against the fetched body (get `snapshotToken` + `data-local-id`s from the fetch). Tested pattern, in a Page Properties table:

```
edits: [
  {"name":"replaceNode","arguments":{},"localId":"<id of the <p> in the Status value cell>","value":"<p><span data-type=\"status\" data-color=\"green\" data-status-style=\"bold\">APPROVED</span> </p>"},
  {"name":"replaceNode","arguments":{},"localId":"<id of the <p> in the Approved by value cell>","value":"<p>Name</p>"},
  {"name":"replaceNode","arguments":{},"localId":"<id of the <p> in the Approved and locked date value cell>","value":"<p><time datetime=\"YYYY-MM-DD\">D Mon YYYY</time></p>"}
]
```

Include the third edit only if the fetched header table actually has an "Approved and locked date" row (Manual / Policy pages); drop it for Development & Testing and Controlled File pages.

- Target the **`<p>` inside the value cell, not the `<td>`** — replacing a `<td>` with a `<p>` deletes the cell. Run once with `dryRun: true` and compare the whole echoed body (not just the table) with the original before the real call.
- `<time>` is re-rendered as a long date ("October 31, 2026").
- Re-fetch afterwards and confirm.

---

## Wrapping an old plain header table in Page Properties (done 2026-10-31 on 12 DV drafts)

Pages cloned from the Manual/Policy template before it gained the macro have a plain header `<table>` (use the `data-local-id` from your own fetch — on the 2026-10-31 batch it was `b24120e1-0142-4420-acd0-4401adc77d8f`, but don't assume it). To wrap one: fetch (html, `detail: "full"`), then a single `replaceNode` on the **table's own `data-local-id`** whose value is `<div data-type="bodied-extension" data-extension-key="details" data-extension-type="com.atlassian.confluence.macro.core" data-layout="default" data-parameters="{&quot;macroParams&quot;:{},&quot;macroMetadata&quot;:{&quot;schemaVersion&quot;:{&quot;value&quot;:&quot;1&quot;},&quot;title&quot;:&quot;Page Properties&quot;}}">` + the table copied verbatim + `</div>`; `dryRun` first, then the real call with `minorEdit: true`.

Lessons:
- Copy `&nbsp;` as an entity — retyping it as a space silently changes the cell (caught in the dry-run echo three times).
- Never fetch in parallel with the write; one pilot got a spurious `internal_error` ("row count 0") that cleared on a sequential retry.
- Expect harmless server re-serialisation outside the edit: `<strong>`/`<span>` nesting order, entity forms, regenerated figure `data-local-id`s, `_parentId` on macros. Occasionally a trailing space in an inline `<code>` span is lost, and one page lost a table's `data-layout="center"` — check long pages visually.
- Use `CQL ... AND NOT macro = "details"` to find pages still lacking the macro; `macro = "details"` works as a filter.

---

## Document-Type Field Reference

Quick reference for the controlled document header fields present in each document type.

**Development & Testing**: Status | Jira ticket | Driver | Navigator | Approved by

**Controlled File**: Status | Jira ticket | Authors | Approved by

**Manual / Policy**: Status | Version | Jira ticket | Author | Reviewed by | Approved by | Approved and locked date | Review schedule | Deactivated Date

---

## Post-Update Reminders

After a sign-off update, remind the user of any remaining manual steps that can't be done via the API:

- **Page permissions**: restrict editing to B8_group (Confluence UI — Page settings → Restrictions)
- **Archiving previous version** (policy/manual only): move old page to Archived folder, prepend `[ARCHIVED]` to its title, fill in its Deactivated Date field — all doable via `confluence-docs update` targeting the old page
- **Create next DRAFT** (policy/manual only): use `confluence-docs create policy-doc` to start the next version
- **Jira ticket**: close the ticket; ensure Prod Comments are populated with What's New content before closing (triggers the egg-announce Slack notification)
- **DI-2023 next-version ticket** (policy/manual only): create a new ticket linked to DI-2023 for the next annual review
