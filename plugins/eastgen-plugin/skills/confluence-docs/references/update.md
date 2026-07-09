# Update — Editing Existing Pages

Use this reference when the mode is `update`. Follow these steps in order.

`updateConfluencePage` replaces the entire page body — there's no partial-update endpoint. The safe approach for surgical edits is:

**Fetch the full HTML+ body → change only the target text → send the full body back.** Always exactly two tool calls, regardless of how small the change.

---

## Step 1: Identify the Target Page

Ask for anything not provided in the arguments:

- **Page URL or page ID**: the page to be edited
- **What needs changing**: a clear description of the edit — e.g. "update Status to APPROVED and add Approved by Katherine Winfield", "add a new test scenario under Testing", "correct the Jira ticket link"

Extract the page ID from the URL if given: `https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/1234567890/...` → `1234567890`.

---

## Step 2: Inspect the Body First

**Always fetch and read the body before writing any edit.** You cannot safely target text without knowing the page's actual current content.

```
getConfluencePage(cloudId, pageId, contentFormat: "html")
```

Read the returned `body` to:
- Confirm the exact text and surrounding markup of what you intend to change
- Check whether a given text value appears more than once (e.g. "N/A" may appear in many cells — include enough surrounding context to make the target unique, the same discipline as any exact-match text edit)
- Identify table structure (how many rows, what labels are in the first column) before editing a specific cell

Only write the edit once you've seen the actual body.

---

## Step 3: Make the Edit and Send It Back

Change only the specific span of HTML that needs to change — keep everything else byte-for-byte as fetched — then call:

```
updateConfluencePage(cloudId, pageId, title, body, contentFormat: "html")
```

Pass the existing title unchanged unless the title itself is part of the edit.

**Safety rules:**
- Include enough surrounding markup that the targeted text is uniquely identified before changing it
- Change only the specific nodes that need to change, keeping everything else — including `data-local-id` attributes — verbatim
- Re-fetch after the update and confirm the change landed as intended

---

## Common Update Scenarios

### Sign-off: update the controlled document header

Applies to all document types. Updates the status lozenge, Approved by cell, and Approved and locked date cell.

Fetch the body, then apply all three changes before sending it back once:

1. Status lozenge → `<span data-type="status" data-color="yellow">DRAFT</span>` becomes `<span data-type="status" data-color="green">APPROVED</span>`
2. "Approved by" table cell → replace its `<td>` contents with the approver's name
3. "Approved and locked date" table cell → replace its `<td>` contents with the date

### Update the Jira ticket link

Find the `<a href="https://cuhbioinformatics.atlassian.net/browse/DI-XXXX">DI-XXXX</a>` and replace both the visible text and `href` with the new ticket.

### Rewrite the content under a heading

Replace everything between the target `<h1>`/`<h2>` and the next heading of the same or higher level (e.g. rewriting a test scenario narrative, updating the Changes section for a new version).

### Add a new test scenario section

Insert a new `<h2>` block plus its content immediately before the next `<h1>` (or at the end of the document if there isn't one) — don't touch the existing scenarios.

### Update the page title (e.g. remove DRAFT)

Pass the new title in the same `updateConfluencePage` call as any body change — no need for a separate call.

### Update a single table cell by label

Find the `<tr>` whose first `<td>`/`<th>` matches the label, then replace the contents of its second cell.

### Add a row to a table

Insert a new `<tr>` immediately before the table's closing `</tbody></table>`.

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
