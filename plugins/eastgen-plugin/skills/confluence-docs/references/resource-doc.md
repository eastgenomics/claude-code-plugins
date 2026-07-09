# Controlled File Doc — Creation Guide

Use this reference when the mode is `create resource-doc`. Follow these steps in order.

Historical template ID (not used at runtime — see SKILL.md → Access): `3198255618`.

---

## Step 1: Gather Inputs

Ask for anything not provided in the arguments:

- **Title**: exact resource name (e.g. `TWE38_POPAF_chr1-22_241126.vcf.gz`, `haemonc_panel_v3_sentieon_250811_260413`)
- **Parent page URL**: Confluence page this should be a child of
- **Jira ticket**: associated ticket URL (`DI-XXXX` or `EBH-XXXX`)
- **Authors**: who was involved in creating and documenting this resource
- **Approved by**: leave blank if not yet known
- **Resource version**: e.g. `v1`
- **DNAnexus file ID**: if uploaded to DNAnexus — omit the row entirely if not applicable
- **Content summary**: what data is in the resource (e.g. "Population allele frequencies for 35 TWE runs, 1444 samples, GRCh38")
- **Non-functional summary**: file format, size, index files, access mechanism (e.g. "VCF.gz with .tbi index", "Apache Iceberg table queryable via AWS Athena")
- **What's new**: differences from previous version, or "First version — not applicable"
- **Reference / publication**: links to papers, GitHub repos, AWS docs, prior Confluence pages
- **Suggested use cases**: which tools, workflows, or pipelines will use this resource
- **Provenance**: how the resource was created — source data, steps taken, commands run, date ranges
- **Testing performed**: what was done to validate the resource
- **Overall Outcome**: pass/fail recommendation, or "To be completed" if not ready

---

## Step 2: Build and Create the Page

Resolve the parent page ID from its URL (see SKILL.md → Shared conventions), then build the complete HTML+ body per the Content Guide below and create the page in one call:

```
createConfluencePage(cloudId, spaceId, parentId, title, body, contentFormat: "html")
```

Report the returned page's URL.

---

## Content Guide

### Page Properties table
Status (leave as DRAFT, yellow lozenge), Jira ticket (link), Authors, Approved by.

### H1: Controlled file summary
A table with these rows — replace each placeholder cell:

| Field | Content |
|---|---|
| Name | Resource name exactly as used in production |
| Version | e.g. `v1` |
| DNAnexus file ID | File ID — omit this row entirely if not on DNAnexus |
| Content summary | What data is in the resource; sample/variant/record counts |
| Non-functional summary | File format, size, indexing, access mechanism |
| What's new? | Differences from previous version, or "First version — not applicable" |
| Reference / publication | Links to papers, GitHub, AWS docs, prior Confluence pages |
| Suggested & tested use cases | What tools/workflows will use this, what was tested |

### H1: Overall Outcome
Short paragraph: is the resource fit for purpose? Example:
> "This variant store has been created from 2,542 normalised MYE VCFs and passes data integrity checks. It is recommended for use as a source for population allele frequency annotation in the haemonc pipeline."

### H1: Provenance
Bullet points and code blocks covering:
- Source data (what went in, date range, counts, origin)
- Steps taken (tools, pipelines, S3 locations, AWS job IDs)
- Any filtering, deduplication, or QC applied to inputs
- Commands run (code block, `bash`)

### H1: Testing
Start with a Testing Summary table (Test | Outcome columns). Then one H2 per test scenario:

- **Testing Strategy**: what was tested and why
- **Expected Outcome**: what a passing result looks like
- **Commands used**: code block
- **Test Output**: describe or paste the output
- **Test Outcome**: status lozenge (green = PASS, red = FAIL, yellow = Pass with warning) + narrative

---

## Testing Guidance

Help the user frame their testing appropriately:

- **Data integrity**: record counts, sample counts, file checksums, index validity
- **Content spot checks**: query known variants/samples and verify expected values
- **Provenance verification**: confirm input counts match output counts
- **Format validation**: file can be opened/queried by the intended tool
- **Comparative testing**: compare against previous version or external reference for a subset
- **Boundary/edge cases**: samples with 0 reads, chrM variants, multi-allelic sites

---

## After Creation

Report the page URL. Remind the user:
- Fill in Approved by when ready
- Complete the Overall Outcome section if not done
- To sign off: update the status lozenge to APPROVED, add Approved by + date, lock the page (Anyone can view, B8_group can edit) — use `confluence-docs update` for these changes
