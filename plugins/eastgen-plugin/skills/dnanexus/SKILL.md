---
name: dnanexus
description: DNAnexus cloud genomics platform for East Genomics — building/deploying apps and applets, ad-hoc jobs via app-swiss-army-knife, dx/dxpy data operations, job and workflow monitoring, and the Jira/GitHub/Confluence release process for DNAnexus work. Use for dx CLI commands, dxapp.json, app-swiss-army-knife, or any org-emee_1 DNAnexus task.
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, mcp__plugin_eastgen-plugin_dnanexus-documentation__searchDocumentation, mcp__plugin_eastgen-plugin_dnanexus-documentation__getPage, mcp__plugin_eastgen-plugin_dnanexus-documentation__askQuestion
---

# DNAnexus Integration (East Genomics)

## Overview

DNAnexus is a cloud platform for biomedical data analysis running on AWS `eu-central-1`. This skill covers building and deploying apps/applets using the organisation's standard patterns, managing data objects, and scripting with the dxpy Python SDK. Requires a DNAnexus account with access to `org-emee_1` — see **Authentication** below for the required account type.

## Organisation Context

- **Org**: `org-emee_1`
- **Region**: `aws:eu-central-1`
- **App naming convention**: `eggd_<name>` (e.g. `eggd_generate_variant_workbook`)
- **Standard Ubuntu release**: `24.04` (updated from 20.04; requires `runSpec.version: "0"` in dxapp.json). The org's own `DNAnexus_app_template` repo still ships `20.04` — treat that as stale, not as guidance; use `24.04` for new apps.
- **Default instance type**: `mem1_ssd1_v2_x4`
- **App template repo**: [DNAnexus_app_template](https://github.com/eastgenomics/DNAnexus_app_template) — start new app repos from this.
- **DNAnexus project prefixes**: `001_` reference data, `002_` production clinical service runs, `003_` general working/dev projects (auto-archived/deleted over time), `004_` validation projects with compliance-retained data. Dev/test work: `003_YYMMDD_<kebab-case-topic>`; rename to `004_...` if results need long-term retention. See `references/development-lifecycle.md`.
- **Production release must be an app, not an applet**: `dx build --app`, then `dx publish eggd_x/version`. Applets are fine for `003_`/`004_` development/testing only.

## Authentication

Authenticate non-interactively with `$DNANEXUS_API_TOKEN` — `dx login --token "$DNANEXUS_API_TOKEN" --noprojects` (CLI) or dxpy's `dxpy.set_security_context({"auth_token_type": "Bearer", "auth_token": os.environ["DNANEXUS_API_TOKEN"]})` — never an interactive password login, and never hardcode the token value.

**This token must belong to a dedicated agent/service DNAnexus account** (`MEMBER` org role, `CONTRIBUTE` project permission — never `ADMINISTER`) — never a person's own DNAnexus login. Check 3 in "Verifying the account is actually restricted" (below) confirms the account can't delete in one purpose-built `protected` test project; it doesn't enumerate every project the account has access to, so a stray project-scoped `ADMINISTER` grant elsewhere wouldn't be caught by it — see also the note in **File ID Resolution** below.

**What "delete disabled" actually means for this account — two separate mechanisms, don't conflate them:**
- **Org-level `dataDeletion: Not Allowed`** (set on the account) blocks only the org-wide bypass (`overrideProjectAccess: true` — deleting in *any* project regardless of membership). It does **not** block ordinary deletion in a project the account is a normal `CONTRIBUTE` collaborator on.
- **The actual delete gate is per-project**: a project's `protected` flag. `CONTRIBUTE` can delete whenever `protected` is `false` (the default); only `ADMINISTER` can delete when `protected` is `true`. This has been verified empirically, not just from docs — the agent account successfully deleted files (both one it uploaded and a pre-existing one) in an unprotected (`protected: false`) test project with no resistance at all.
- **By deliberate policy, this is not applied uniformly across project tiers**: `003_` projects are disposable scratch space by design (already subject to their own auto-archive/delete lifecycle), so they are intentionally left unprotected — the agent account genuinely can delete there, and that's expected, not a bug. `001_`/`002_`/`004_` are expected to have `protected: true` (confirmed separately for `001_`/`002_`; `004_` — compliance-retained validation data — needs the same check). If you ever need to confirm whether delete is actually blocked in a specific project, check that project's own `protected` flag (`dx describe project-xxx --json | jq .protected`) — don't assume from the account's org-level settings alone.

Account setup is a one-off human task (Claude cannot fetch this page itself): [Onboarding for Claude](https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/4739137538/Onboarding+for+Claude) (Confluence).

Before running anything against real data, confirm `dx whoami` resolves to the agent account, not a personal one — see the Common Gotchas entry below.

### Verifying the account is actually restricted, after setup or rotation

None of these three checks alone rules out every misconfiguration — they check different,
independent axes, run all three:

1. **Org-level**: `dx api org-emee_1 findMembers '{"id": ["user-<agent>"]}'` (source:
   [Organizations](https://documentation.dnanexus.com/developer/api/organizations)) shows
   this member's `level` as `MEMBER` with `dataDeletion: false` ("Not Allowed") — **not**
   `ADMIN`. (`dx find org members org-emee_1 --json` lists everyone if you need to find the
   exact user ID first, but only filters by level, not by member — use `findMembers` to
   target one account directly.) This only rules out the org-wide bypass axis; it says
   nothing about the account's permission on any specific project.
2. **Identity**: `dx whoami` shows the agent account, authenticated via
   `dx login --token "$DNANEXUS_API_TOKEN"` (not a personal login).
3. **Project-level delete test — use a dedicated scratch project created specifically for
   this check, never `001_`/`002_`/`004_` or any real project.** Using your own
   admin-capable login: create the project, upload one disposable object into it, *then*
   set `protected: true` and invite the agent account as `CONTRIBUTE` — in that order, so
   an untested "can `CONTRIBUTE` even upload to a `protected` project?" question can't be
   confused with the delete check itself. Switch to the agent and attempt to delete that
   object; confirm it gets a permission error. If the delete succeeds, check what level the
   agent actually has *on this specific test project* —
   `dx describe project-xxx --json | jq '.level'`, run **as the agent** (the caller's own
   effective permission — the same `ADMINISTER`/`CONTRIBUTE` vocabulary as the per-project
   values in **File ID Resolution**'s `listProjects` output below, though returned here as
   a single top-level `level` field rather than a project-ID map; the full per-member
   `permissions` map on a project describe generally isn't returned to a `CONTRIBUTE`-level
   caller, which is why `.level` and not `.permissions` is the field to check here) — a
   project-scoped `ADMINISTER` grant is invisible to check 1,
   which only sees the org axis, so a successful delete here isn't explained by check 1
   having already ruled out `ADMINISTER`.

## Quick Task Guide

This table is the router — resolve the task to a row, then go straight to that reference rather than searching this file further. The "Done when" column is the completion bar: don't consider the task finished until it is met, and never treat a folder listing or a wrapper exit code as proof of the intended outcome.

| Task | Tool | Reference | Done when |
|---|---|---|---|
| Submit a quick ad-hoc job (no app build) | `app-swiss-army-knife` | `references/swiss-army-knife.md` | Job is `done` and its **specific** output object (by ID, not folder listing) is verified |
| Build a full reusable app | `eggd_` bash+Python pattern | `references/app-development.md` | Platform test reaches `done` with every declared output present at the expected type/content |
| Configure `dxapp.json` | `inputSpec`/`runSpec`/`assetDepends` | `references/configuration.md` | Config builds and its input/output contract is exercised on the platform |
| Upload/download/find files | `dx upload` / `dx download` / `dx find data` | `references/data-operations.md` | Every input/output is identified by its canonical `project-xxx:file-xxx` reference |
| Monitor jobs, chain jobs, trace provenance | `dx watch`, `dx describe`, dxpy `DXJob` | `references/job-execution.md` | Terminal state established; on failure, the **first causal** error (not a dependency termination) is identified |
| Automate with Python outside a job | `dxpy` SDK | `references/python-sdk.md` | Script uses the intended project context and verifies results |
| Navigate the Jira/GitHub/Confluence dev process | Driver/Navigator/Approver, GitFlow, Story lifecycle | `references/development-lifecycle.md` | Required release, test-evidence, and deployment records exist |
| Raise the PR / respond to review comments | GitHub Flow, Jira-link guardrail | the `pr-workflow` skill (this plugin) | PR raised with the Jira key present; all review comments resolved |
| Write up app testing evidence in Confluence | Documentation Vault dev-doc template | the `confluence-docs` skill, mode `create dev-doc` (this plugin) | Signed-off page describes the exact deployed version |
| Set up (or rotate) the agent DNAnexus account/token — **human-performed, not something Claude does for itself** | New DNAnexus login + org invite | [Onboarding for Claude](https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/4739137538/Onboarding+for+Claude) (Confluence) | See **Authentication** → "Verifying the account is actually restricted" — all three independent checks pass |
| A job/download fails with `InvalidState` or a 422 error | Check `archivalState` on the file | `## File Archival State` (below) | File is `live` (or successfully unarchived) and the job/download re-run succeeds |
| Look up official DNAnexus platform behaviour (not this org's conventions) | `dnanexus-documentation` MCP (`askQuestion`/`searchDocumentation`/`getPage`) | "Getting Help", below | Answer is sourced to a doc page; the query sent contained no org-internal identifiers |

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

**Under the restricted agent account** (see **Authentication**, above): `listProjects`
should never show `ADMINISTER` for any project, since the agent account is meant to be
capped at `CONTRIBUTE` — if it does, that's a misconfiguration (the same one "Verifying
the account is actually restricted" checks for), not a sign that the heuristic below is
wrong; escalate it rather than treating the `ADMINISTER` entry as the canonical answer.
Ordinarily, though, this heuristic can't identify the canonical project from permissions
alone for this account — fall back to the project the file was originally uploaded/generated
in (from job/upload records), or ask a human with `ADMINISTER` access to confirm.

This resolution step applies everywhere a file ID shows up below — job inputs, uploads, dxpy calls — not just here. Exception: an API method that takes the project as a separate positional argument (e.g. `dx api project-xxx unarchive '{"files": [...]}'`, where the file IDs in the array are inherently scoped to the project already given as the call's target) doesn't need the IDs inside it additionally qualified.

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

**Under the restricted agent account** (`CONTRIBUTE`, no `ADMINISTER` — see **Authentication**,
above): confirmed — `CONTRIBUTE` is sufficient for `/project-xxxx/unarchive`; it is not
gated behind `ADMINISTER` or a project's `protected` flag at all, which is a separate
permission (source: [Project Permissions and Sharing](https://documentation.dnanexus.com/developer/api/data-containers/project-permissions-and-sharing)).
The clone-then-unarchive fallback (step 2 above) is unnecessary for this account — unarchive
directly in place instead of cloning first.

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
    -iin="project-xxxx:file-aaaa" -iin="project-xxxx:file-bbbb" \
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
│   ├── home/dnanexus/
│   │   ├── myapp/          # Python source (deployed to /home/dnanexus/myapp/)
│   │   │   └── myapp.py    # Pure Python CLI — no dxpy imports
│   │   └── packages/       # Bundled .whl files (offline pip install)
│   └── usr/local/bin/
│       ├── mark-section    # Structured logging utility
│       └── mark-success    # Job success marker
└── requirements.txt        # Python dependencies (for local dev reference)
```

The `resources/` directory is **overlaid onto the execution filesystem** at build time — no runtime path resolution needed.

Production release of an app must go through `dx build --app` + `dx publish` and satisfy the org's code-review checklist (app not applet, `eggd_` prefix, `org-emee_1`-only access, `aws:eu-central-1`, timeout set, `assetDepends` preferred over manual installs, `set -e` minimum, pinned deps). See `references/development-lifecycle.md` for the full checklist and the Jira/GitHub process around it.

**Under the restricted agent account** (see **Authentication**, above): confirmed — `/app-xxxx/publish` is gated by app-creator/developer authorization, not project `ADMINISTER` or org role (source: [Apps](https://documentation.dnanexus.com/developer/api/running-analyses/apps)). So the agent account can `dx build --app` + `dx publish` **only for an app it built itself** (making it the creator) — it has no publish rights over an app someone else (a human, or a different account) created, even with `CONTRIBUTE` on the project. If the agent is asked to publish an app it didn't build, that will fail with a permission error; escalate to whoever built it (or have the agent build it in the first place) rather than trying to work around it.

---

## Common Gotchas

These cut across app development, swiss-army-knife jobs, and job execution alike — check this list before assuming a failure is a code bug.

### Confirm you're authenticated as the agent account, not a personal one

A stale shell session, a wrong/unset env var, or a leftover interactive `dx login` can
silently leave `dx` authenticated as a person's own DNAnexus account instead of the
restricted agent account — which has full personal permissions, including `ADMINISTER`
on projects and any org-wide delete override, not just the capped `CONTRIBUTE` access
the agent account is meant to have. Check before running anything against real data:

```bash
dx whoami
# Must NOT match the person's own personal DNAnexus username — by convention the
# agent account is a visibly distinct login (e.g. ends in _agent, per Onboarding
# for Claude). If you don't know the expected agent username for this environment,
# ask rather than guessing whether the printed name "looks like" an agent account.
```

If it matches (or might match) a personal account, stop — don't re-run
`dx login --token "$DNANEXUS_API_TOKEN" --noprojects` and assume that fixes it, since a
wrong result usually means the env var itself holds the wrong token, and re-running the
same login just reproduces the same wrong identity. Re-check `dx whoami` after any fix
attempt; if it's still wrong, escalate to a human rather than retrying or proceeding.

**`dx whoami` only verifies the CLI session — it does not prove `dxpy.set_security_context(...)`
is configured with the same token.** The CLI (`dx`) and dxpy's security context are set
independently; a dxpy-based script (see `references/python-sdk.md`, `references/data-operations.md`)
can be running under a different identity than whatever `dx whoami` shows. For dxpy/Python
automation, verify identity in that same context instead — call `dxpy.api.system_whoami()`
right after `dxpy.set_security_context(...)` and confirm it resolves to the agent account
before proceeding, the same way `dx whoami` is checked for CLI work.

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

**Under the restricted agent account** (see **Authentication**, above): confirmed —
`--overwrite`/`-f` explicitly means "remove existing applet(s) of the same name in the
destination folder" (source: [Index of dx commands](https://documentation.dnanexus.com/user/helpstrings-of-sdk-command-line-utilities)),
i.e. it performs a delete, which is gated by the destination project's own `protected`
flag (see **Authentication**, above) — **not** by anything set on the account itself.
In the common case (`003_` dev/test projects, which are deliberately left unprotected),
`--overwrite` works exactly as documented — this is the normal, expected path for applet
development. It only fails if the destination happens to be a genuinely `protected: true`
project (`001_`/`002_` are confirmed to be; check before assuming `004_` is too — see
**Authentication**), in which case both `--overwrite` and a plain rebuild against an
existing same-named applet fail the same way (no delete rights) — escalate to a human
with `ADMINISTER` there rather than working around it, since applet rebuilds targeting a
protected project are the exception, not the norm.

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
output file ID, never assume the newest by folder listing). The gotcha that **the BAI index must sit alongside the BAM** is documented the same way in `references/swiss-army-knife.md`.

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

- **Official documentation is queryable directly** via the bundled `dnanexus-documentation`
  MCP server — use `askQuestion` for a direct, sourced answer (permission models, API
  behaviour, anything this skill doesn't cover or might be stale on), `searchDocumentation`
  to browse matching pages, and `getPage` to read one in full. Prefer this over guessing
  when a question is about official DNAnexus platform behaviour rather than this org's own
  conventions. **Every query sent to this server is free text delivered to a third party
  (GitBook/DNAnexus) — never include org-internal identifiers in it**: no
  `project-xxx`/`file-xxx`/`job-xxx` IDs, no `org-emee_1` project or app names, no
  patient/sample data. Describe things in the abstract (what you observed, not which real
  object it happened to) — DNAnexus's own generic placeholders (`project-xxxx`, `file-xxxx`)
  are the right level of detail, not this org's real identifiers. If you find the docs
  themselves wrong, outdated, or missing something, `sendFeedback` can report it — that tool
  is deliberately **not** in this skill's pre-authorized `allowed-tools`, so invoking it goes
  through the normal permission prompt instead of skipping it; don't work around that by
  asking the user to invoke it in your place.
- Official documentation: https://documentation.dnanexus.com/
- dx-toolkit GitHub: https://github.com/dnanexus/dx-toolkit
- Agent account setup (email alias, org invite, token generation): [Onboarding for Claude](https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/4739137538/Onboarding+for+Claude) (Confluence, human-only — not fetchable by Claude)
