# Policy / Manual Doc — Creation Guide

Use this reference when the mode is `create policy-doc`. Follow these steps in order.

Historical template ID (not used at runtime — see SKILL.md → Access): `4124934145`.

Policy and manual documents live in the **Operational Guidance** section of the Documentation Vault. They cover processes, policies, and operational procedures (e.g. training and competency policies, run analysis manuals, operational guides) — distinct from Development & Testing and Controlled File documents.

---

## Step 1: Gather Inputs

Ask for anything not provided in the arguments:

- **Title**: name and integer version (e.g. `Bioinformatics Training and Competency Policy v4`, `Helios Run Analysis Manual v2`)
- **Parent page URL**: Confluence page this should be a child of (usually a folder inside Operational Guidance)
- **Jira ticket**: associated `DI-XXXX` URL
- **Author**: who authored this version
- **Reviewed by**: reviewer name(s), or "N/A" if not applicable
- **Approved by**: leave blank if not yet known
- **Version**: integer version number (e.g. `4`)
- **Review schedule**: default `Annual`
- **What's new**: bullet list of changes from previous version, or "First version — not applicable"
- **Purpose**: what this document covers, who it applies to, and any regulatory or standards drivers (e.g. ISO 15189:2022, CU-TR-POL-2)
- **Useful linked documentation**: related documents — each entry needs a link and a comment describing its relevance
- **Content sections**: the main body — ask the user to describe each section heading and its content (freeform; see common patterns below)

---

## Step 2: Build and Create the Page

Resolve the parent page ID from its URL (see SKILL.md → Shared conventions), then build the complete HTML+ body per the Content Guide below and create the page in one call:

```
createConfluencePage(cloudId, spaceId, parentId, title, body, contentFormat: "html")
```

Report the returned page's URL.

---

## Content Guide

### Controlled document header table

Always the first table on the page. Fill these cells:

| Field | Notes |
|---|---|
| Status | Leave as `DRAFT` (yellow lozenge) on creation |
| Version | Integer, e.g. `4` |
| Jira ticket | Link to `DI-XXXX` |
| Author | Author name |
| Reviewed by | Reviewer name, or `N/A`; omit this row if entirely not applicable |
| Approved by | Leave blank if not yet known |
| Approved and locked date | Leave blank on creation |
| Review schedule | Default: `Annual` |
| Deactivated Date | Leave blank on creation |

### H1: What's new?
Bullet list of changes from the previous version. First version: "First version — not applicable."

### H1: Purpose
Prose explaining what the document covers, who it applies to, and any standards/regulatory drivers.

### H1: Useful linked documentation
Table, two columns: **Document** (link) | **Comment** (relevance description). Leave empty with a placeholder paragraph if none provided.

### H1+: Content sections (user-defined)
Remaining sections are determined by the policy content. Common patterns:

- **Roles & responsibilities** — table: Role | Description | Responsibilities
- **Training process / Workflow** — ordered list or narrative with decision logic
- **Frequency / Criteria** — bulleted rules and thresholds
- **Categories / Types** — table mapping types to templates, evidence requirements, assessors
- **Withdrawal / Exceptions** — prose + short example table
- **Appendix / Template guidance** — nested H2 subsections

Use H1 for top-level sections, H2 for subsections.

---

## Versioning Conventions

- Policy/manual versions use integers (v1, v2, v3…), not semantic versioning
- Each policy has a **live** locked version and a **DRAFT next version** in progress
- New versions include `DRAFT` in the title until approved (removed at sign-off)
- The previous version is archived: moved to the Archived folder, title prepended with `[ARCHIVED]`, Deactivated Date filled in

---

## After Creation

Report the page URL. Remind the user:
- Fill in Reviewed by / Approved by / Approved and locked date when review is complete
- Copy the What's New content to the Prod Comments field on the Jira ticket before closing it (triggers the automated egg-announce Slack notification)
- Create a new ticket in DI-2023 for the next version's review cycle (automation sets the due date 365 days ahead)
- To sign off: update Status to APPROVED, remove DRAFT from the title, lock the page (Anyone can view, B8_group can edit) — use `confluence-docs update` for these changes
- Create a new `DRAFT` child page for the next version
- Archive the previous version: move to Archived folder, prepend `[ARCHIVED]` to its title, fill in Deactivated Date
