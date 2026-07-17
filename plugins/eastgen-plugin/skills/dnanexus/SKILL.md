---
name: dnanexus
description: DNAnexus cloud genomics platform for East Genomics — building/deploying apps and applets, ad-hoc jobs via app-swiss-army-knife, dx/dxpy data operations, job and workflow monitoring, and the Jira/GitHub/Confluence release process for DNAnexus work. Use for dx CLI commands, dxapp.json, app-swiss-army-knife, or any org-emee_1 DNAnexus task.
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# DNAnexus Integration (East Genomics)

## Overview

DNAnexus is a cloud platform for biomedical data analysis running on AWS `eu-central-1`. This skill covers building and deploying apps/applets using the organisation's standard patterns, managing data objects, and scripting with the dxpy Python SDK. Requires a DNAnexus account with access to `org-emee_1`.

## Organisation Context

- **Org**: `org-emee_1`
- **Region**: `aws:eu-central-1`
- **App naming convention**: `eggd_<name>` (e.g. `eggd_generate_variant_workbook`)
- **Standard Ubuntu release**: `24.04` (updated from 20.04; requires `runSpec.version: "0"` in dxapp.json). The org's own `DNAnexus_app_template` repo still ships `20.04` — treat that as stale, not as guidance; use `24.04` for new apps.
- **Default instance type**: `mem1_ssd1_v2_x4`
- **App template repo**: [DNAnexus_app_template](https://github.com/eastgenomics/DNAnexus_app_template) — start new app repos from this.
- **DNAnexus project prefixes**: `001_` reference data, `002_` production clinical service runs, `003_` general working/dev projects (auto-archived/deleted over time), `004_` validation projects with compliance-retained data. Dev/test work: `003_YYMMDD_<kebab-case-topic>`; rename to `004_...` if results need long-term retention. See `references/development-lifecycle.md`.
- **Production release must be an app, not an applet**: `dx build --app`, then `dx publish eggd_x/version`. Applets are fine for `003_`/`004_` development/testing only.

## Quick Task Guide

This table is the router — resolve the task to a row, then go straight to that reference rather than searching this file further:

| Task | Tool | Reference |
|---|---|---|
| Submit a quick ad-hoc job (no app build) | `app-swiss-army-knife` | `references/swiss-army-knife.md` |
| Build a full reusable app | `eggd_` bash+Python pattern | `references/app-development.md` |
| Configure `dxapp.json` | `inputSpec`/`runSpec`/`assetDepends` | `references/configuration.md` |
| Upload/download/find files | `dx upload` / `dx download` / `dx find data` | `references/data-operations.md` |
| Monitor jobs, chain jobs, trace provenance | `dx watch`, `dx describe`, dxpy `DXJob` | `references/job-execution.md` |
| Automate with Python outside a job | `dxpy` SDK | `references/python-sdk.md` |
| Navigate the Jira/GitHub/Confluence dev process | Driver/Navigator/Approver, GitFlow, Story lifecycle | `references/development-lifecycle.md` |
| Raise the PR / respond to review comments | GitHub Flow, Jira-link guardrail | the `pr-workflow` skill (this plugin) |
| Write up app testing evidence in Confluence | Documentation Vault dev-doc template | the `confluence-docs` skill, mode `create dev-doc` (this plugin) |

---

## File ID Resolution — Critical

A DNAnexus file ID (e.g. `file-Gb73P8Q4XGyX9QZ0g9qp61xV`) is globally unique but can
exist in **multiple projects simultaneously** as the same underlying object. `dx describe
file-xxx` (without a project qualifier) returns whichever project is found first in the
current context — which is often **not** the canonical/authoritative project.

**Always use `listProjects` when given a bare file ID:**

```bash
dx api file-Gb73P8Q4XGyX9QZ0g9qp61xV listProjects
# Returns:
# { "project-Fkb6Gkj433GVVvj73J7x8KbV": "ADMINISTER",   ← canonical home
#   "project-Gb7329Q4XGyX6yz59fvp0V5j": "CONTRIBUTE" }  ← reference copy
```

The project with `ADMINISTER` permission is the canonical source.
`CONTRIBUTE`-only projects hold reference copies. Always record canonical IDs as
`project-xxx:file-xxx` qualified strings — never bare `file-xxx` alone.

This resolution step applies everywhere a file ID shows up below — job inputs, uploads, dxpy calls — not just here.

---

## File Archival State

Files can be **archived** to reduce storage costs. Archived files cannot be downloaded
and will cause jobs to fail with `InvalidState` / code 422.

```bash
# Check archival state
dx describe project-xxx:file-xxx --json | \
    python3 -c "import sys,json; print(json.load(sys.stdin).get('archivalState','?'))"
# States: live | archival (unarchiving in progress) | archived

# Unarchive in a project you ADMINISTER
dx api project-xxx unarchive '{"files": ["file-aaa", "file-bbb"]}'

# If the file is in a project you don't ADMINISTER:
#   1. Clone it:     dx cp source-project:file-xxx dest-project:/folder/
#   2. Unarchive:    dx api your-project unarchive '{"files": ["file-xxx"]}'
# Small files: seconds. Large FASTA (~800 MB): up to 15 min.
```

---

## Swiss-army-knife — Ad-hoc Jobs

`app-swiss-army-knife` runs an arbitrary bash command on a DNAnexus worker with
input files automatically downloaded — the fastest way to run a one-off analysis
without building a full app. Full pattern, escaping rules, polling, and instance
types: `references/swiss-army-knife.md`. The two mistakes that hit almost every
first attempt:

```bash
# WRONG — $in_000 / $in_000_path do not exist; find files by glob instead
-icmd="bcftools view \${in_000} > out.vcf"
-icmd='BAM=$(ls *.bam); bcftools view "$BAM" > out.vcf'   # RIGHT

# WRONG — --project and --destination conflict, don't combine them
dx run app-swiss-army-knife --project "project-xxxx" --destination "/folder/" ...
dx run app-swiss-army-knife --destination "project-xxxx:/folder/" ...   # RIGHT
```

```bash
dx run app-swiss-army-knife \
    --destination "project-xxxx:/output/folder/" \
    -iin="file-aaaa" -iin="file-bbbb" \
    -icmd="<bash commands>" \
    --name "my_job" --instance-type mem1_ssd1_v2_x4 \
    -y --brief
```

---

## Primary App Pattern

Apps at this organisation use a **Bash entry point calling a pure Python CLI** — do not assume a Python dxpy entry point. Full `code.sh` pattern, input variable types, and the Python CLI side: `references/app-development.md`.

```
eggd_myapp/
├── dxapp.json              # App metadata, inputs, outputs, run spec
├── src/
│   └── code.sh             # Bash entry point — handles all DNAnexus I/O
├── resources/
│   └── home/dnanexus/
│       ├── myapp/          # Python source (deployed to /home/dnanexus/myapp/)
│       │   └── myapp.py    # Pure Python CLI — no dxpy imports
│       └── packages/       # Bundled .whl files (offline pip install)
│   └── usr/local/bin/
│       ├── mark-section    # Structured logging utility
│       └── mark-success    # Job success marker
└── requirements.txt        # Python dependencies (for local dev reference)
```

The `resources/` directory is **overlaid onto the execution filesystem** at build time — no runtime path resolution needed.

Production release of an app must go through `dx build --app` + `dx publish` and satisfy the org's code-review checklist (app not applet, `eggd_` prefix, `org-emee_1`-only access, `aws:eu-central-1`, timeout set, `assetDepends` preferred over manual installs, `set -e` minimum, pinned deps). See `references/development-lifecycle.md` for the full checklist and the Jira/GitHub process around it.

---

## Common Gotchas

These cut across app development, swiss-army-knife jobs, and job execution alike — check this list before assuming a failure is a code bug.

### `pip install` on Ubuntu 24.04 workers — use a venv

Ubuntu 24.04 enforces PEP 668 (externally managed Python). Plain `pip install` fails
with `RECORD file not found` for packages installed by the OS package manager.
`--break-system-packages` is not always sufficient.

**Always install into a virtual environment on Ubuntu 24.04 workers:**

```bash
# In code.sh or -icmd=
python3 -m venv /tmp/myenv
/tmp/myenv/bin/pip install cnvkit --quiet 2>&1 | tail -3
export PATH="/tmp/myenv/bin:$PATH"
```

This applies to both `app-swiss-army-knife` jobs and custom applets.

### `set -euo pipefail` vs `set -eo pipefail` in applets

`set -u` (undefined variable error) **leaks into the DNAnexus job wrapper** and causes
output upload to fail silently at job end. Use `set -eo pipefail` (no `-u`) in `code.sh`
for all applets and swiss-army-knife job commands.

### `((VAR++))` exit code trap

In bash with `set -e`, `((SUBMITTED++))` when `SUBMITTED=0` evaluates to 0 (falsy)
and triggers an immediate exit. Use the assignment form instead:

```bash
# WRONG — exits when SUBMITTED is 0
((SUBMITTED++))

# RIGHT
SUBMITTED=$((SUBMITTED + 1))
```

### Job tracking — use explicit job ID files, not `dx find jobs --name`

When running multiple batches iteratively, `dx find jobs --name "prefix_*"` returns
**all historical jobs** with that name prefix, including failed jobs from previous runs.
This inflates counts and can trigger downstream steps prematurely (e.g. a build
starting with only 2/41 jobs done because 53 old failed jobs look like "done"). This
is the correct version of the pattern — the `poll_jobs` example in
`references/swiss-army-knife.md` uses name-based `dx find jobs` for brevity; prefer
the tracking-file approach below whenever jobs are submitted iteratively across runs.

**Write job IDs to a tracking file at submission time:**

```bash
# In the submission loop:
JOB_ID=$(dx run ... --brief | tr -d '[:space:]')
echo "${JOB_ID}" >> /tmp/my_batch_jobs.txt

# Poll against specific IDs:
while IFS= read -r JID; do
    STATE=$(dx describe "$JID" --json 2>/dev/null | \
        python3 -c "import sys,json; print(json.load(sys.stdin).get('state','?'))")
    # count states...
done < /tmp/my_batch_jobs.txt
```

Reset the tracking file with `> /tmp/my_batch_jobs.txt` before each new batch submission.

### Job priority — set at submission, cannot change after

Jobs submitted with `--priority normal` may queue for >10 minutes during busy periods.
Use `--priority high` for interactive development and time-sensitive runs.

**Priority cannot be changed after submission.** Terminate and resubmit at the
desired priority if a job is stuck in the runnable queue.

```bash
# Terminate a queued job and resubmit at high priority
dx terminate job-xxxx
dx run ... --priority high ...
```

### Applet build gotchas (Ubuntu 24.04)

This covers `dx build` mechanics common to both applets and apps. Applets built
this way are appropriate for `003_`/`004_` development and testing projects; a
production release must still go through `dx build --app` + `dx publish` per the
org checklist (`references/development-lifecycle.md`).

**`systemRequirements` conflict:** Cannot set `systemRequirements` in both `runSpec`
and `regionalOptions` simultaneously — one or the other, not both.

**`runSpec.version` required for 24.04:** Ubuntu 24.04 requires `"version": "0"` in
`runSpec`. Omitting it gives a 422 error.

**`/applets/` folder must exist before `dx build`:**

```bash
dx mkdir -p "project-xxx:/applets/"
dx build applet_dir/ --destination "project-xxx:/applets/" --overwrite
```

**`dx build --overwrite` creates a new applet ID** each time. Always capture the new
ID from the JSON output and update `resource_ids.env`:

```bash
NEW_ID=$(dx build applet_dir/ --destination "project-xxx:/applets/" \
    --overwrite 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
sed -i "s|export APPLET_FOO=.*|export APPLET_FOO=\"${NEW_ID}\"|" resource_ids.env
```

**Applet `dxapp.json` template fields required by East Genomics** — see `references/configuration.md` for the full spec; the fields specific to East Genomics rather than the DNAnexus default are:

```json
{
  "properties": { "githubRelease": "1.0.0" },
  "developers": ["org-emee_1"],
  "authorizedUsers": ["org-emee_1"],
  "runSpec": { "release": "24.04", "version": "0" }
}
```

### Cross-project file inputs to `dx run`

When passing files from a different project as job inputs, always use
project-qualified IDs. Bare `file-xxx` only resolves in the current project context:

```bash
# WRONG — may resolve to wrong project copy
-iin="file-Gb73P8Q4XGyX9QZ0g9qp61xV"

# RIGHT — always qualify with the canonical project
-iin="project-Fkb6Gkj433GVVvj73J7x8KbV:file-Gb73P8Q4XGyX9QZ0g9qp61xV"
```

This applies to `dx run`, `dx download`, and any dxpy API call.

### `dx find data` — correct syntax

```bash
# WRONG
dx find "myfile.bam"
dx find data "myfile.bam"

# RIGHT
dx find data --name "myfile.bam" --project project-xxxx
dx find data --path "project-xxxx:/folder/" --name "*.bam"
dx find data --name "*.bam" --all-projects   # search across all projects
```

### Chromosome naming and multi-version files — see data-operations.md

Two more gotchas that bite often enough to flag here but are documented in full,
with fix commands, in `references/data-operations.md`: **chromosome naming**
(chr-prefixed vs no-chr BAMs/BEDs/VCFs producing silent empty output on mismatch)
and **multiple file versions at the same path** (always fetch a specific job's
output file ID, never assume the newest by folder listing). The **BAI-index-must-
sit-alongside-BAM** gotcha is documented the same way in `references/swiss-army-knife.md`.

---

## Writing up testing evidence in Confluence

Every DNAnexus app/workflow/config-file release needs a signed-off testing-evidence
page in the CUH Bioinformatics Documentation Vault (space `DV`) before a Jira Story
can move to `SIGNOFF` — this is an ISO 15189 compliance requirement, checked at code
review. Use the **`confluence-docs` skill** (this plugin), mode `create dev-doc`, to
build that page from a live Confluence template — don't build it from scratch. That
mode already knows the DNAnexus-specific shape (job URL patterns, `dx build`/`dx run`
evidence, PASS/FAIL status lozenges per test scenario). See
`references/development-lifecycle.md` for where sign-off fits in the Jira Story
lifecycle, and use the `confluence-docs` skill's `update` mode for edits after
initial creation — the signed-off page must describe the exact version deployed,
never a stale earlier draft.

## References

- `references/development-lifecycle.md` — Jira roles/workflow, GitFlow branching, DNAnexus project prefixes, app "definition of done" checklist, release & deploy steps
- `references/app-development.md` — App structure, `code.sh` patterns, resources overlay, building and deploying
- `references/configuration.md` — Complete `dxapp.json` specification with org-specific fields
- `references/data-operations.md` — `dx` CLI patterns for data I/O, chromosome naming, multi-version files
- `references/job-execution.md` — Running jobs, monitoring, workflows, provenance
- `references/python-sdk.md` — dxpy SDK reference for automation scripts
- `references/swiss-army-knife.md` — Full ad-hoc job reference (escaping, polling, instance types, BAI index)

## Getting Help

- Official documentation: https://documentation.dnanexus.com/
- dx-toolkit GitHub: https://github.com/dnanexus/dx-toolkit
