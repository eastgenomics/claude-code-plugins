# Development & Testing Doc — Creation Guide

Use this reference when the mode is `create dev-doc`. Follow these steps in order.

Clone source: the [Development & Testing live template page](https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/4745560161/Development+Testing+doc+template+copy+this+page+s+content+to+create+a+new+dev-doc) (page ID `4745560161`) — see SKILL.md → Live template pages.

---

## Step 1: Gather Inputs

Ask for anything not provided in the arguments:

- **Title**: name and version of the tool (e.g. `augean v2.0.0`, `eggd_vep v1.3.0`)
- **Parent page URL**: Confluence page this should be a child of
- **Jira ticket**: associated `DI-XXXX` URL
- **GitHub repo or PR links**: used to fetch README, docs/, and PR content
- **Driver / Navigator / Approver**: leave blank if not yet known
- **Changes**: what changed from the previous version (or "First release — not applicable")
- **Overall Outcome**: pass/fail recommendation, or "To be completed" if not ready

---

## Step 2: Fetch GitHub Content

If a GitHub repo or PR link is provided, fetch:

- `README.md` — for the tool summary and purpose
- `docs/` files (`architecture.md`, `testing.md`, `config-guide.md`, etc.) — for design and test details
- The PR page — for commit hash, file counts, and PR description

Use this content to populate the document. Don't ask the user to re-describe work that's already in GitHub.

---

## Step 3: Clone the Live Template and Create the Page

Resolve the parent page ID from its URL (see SKILL.md → Shared conventions), then follow the clone-and-create procedure in SKILL.md → Live template pages, using template page ID `4745560161` and the Content Guide below.

Report the returned page's URL.

---

## Content Guide

### Page Properties table
Status (leave as DRAFT, yellow lozenge), Jira ticket (link), Driver, Navigator, Approved by.

### H1: Summary
What the tool does, why this release exists, any important caveats. Note if any format or feature is non-functional.

### H2: Overall Outcome
Pass/fail recommendation panel. Fill in the outcome, or leave "To be completed."

### H1: Useful linked documentation
Table of `Summary | Link` rows. Include: GitHub repo, architecture.md, config-guide.md, testing.md, Jira ticket.

### H1: Design
Pipeline/architecture overview and key design decisions. Use H2 subsections as appropriate (Pipeline, Config, Database, Data quality handling).

### H1: Changes
Bullet list of changes from the previous version, or "First release — not applicable."

### H1: PRs
Linked PR(s). Each bullet: `PR #N — <title>` linked to the PR URL.

### H1: Test setup
Commit hash, branch, `dx build` command, published app ID (`app-XXXX`), DNAnexus project link, instance type used.

### H1: Testing — required structure

**Reviewers will not sign off without all of the following in every test case.** A flat bullet list of outcomes is insufficient.

**H2: Summary table** — one row per test, with green PASS / red FAIL status lozenges (not ✅ emoji).

**H2 per test: `Tn — <Test name>`** — each test case is its own H2. Every H2 must contain:

1. **Testing strategy** — what was tested, which job/app version, which samples/files.
2. **Expected outcome** — what a passing result looks like.
3. **Provenance of Results** (expand section) — collapsible, containing:
   - **Link to job(s)** — every DNAnexus job ID as a real hyperlink (see URL format below). A plain code span is not sufficient.
   - **Commands used** — the actual `dx run` / `dx describe` / `dx cat` commands in a code block, including the exact app/applet ID, all `-i` inputs, and `--destination`.
4. **Test output** — actual results: job state (`done`), file IDs, observed metric values, code block output from `dx cat` / `dx describe`.
5. **Test outcome** — a green/red status lozenge (PASS / FAIL). Not a text word, not an emoji.
6. **Tip panel** — one-line plain-English outcome summary.

#### DNAnexus URL patterns

Strip the `project-` / `job-` prefix from IDs when building URLs:

```
# Job monitor page
https://platform.dnanexus.com/panx/projects/{PROJECT_ID}/monitor/job/{JOB_ID}
# e.g. project-J8F1vK845FGG0pVG2vfQ4J34  →  J8F1vK845FGG0pVG2vfQ4J34
#      job-J8F9Pgj45FG99zqBFJ6gZ1Kv      →  J8F9Pgj45FG99zqBFJ6gZ1Kv

# Data folder
https://platform.dnanexus.com/panx/projects/{PROJECT_ID}/data/{FOLDER_PATH}
```

#### Reusing the cloned Test Scenario blocks

The cloned template already includes five ready-to-fill `Test Scenario Title` H2 blocks, each with the `Provenance of Results` expand, a status lozenge for Test Outcome, and a tip panel for the summary — reuse that exact shape for every test case rather than building it by hand. If the release has fewer than five tests, delete the extra blocks; if it has more, duplicate one of the existing blocks rather than inventing a new macro shape.

---

## After Creation

Report the page URL. Remind the user:
- Fill in Driver / Navigator / Approved by when ready
- Complete the Overall Outcome section
- To sign off: update the status lozenge to APPROVED, add Approved by + date, lock the page (Anyone can view, B8_group can edit), move the Jira ticket to DEPLOY TO PROD — use `confluence-docs update` for these changes
