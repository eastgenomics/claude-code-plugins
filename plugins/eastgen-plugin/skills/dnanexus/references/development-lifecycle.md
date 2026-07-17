# Development Lifecycle (Jira + GitHub + Confluence)

## Overview

DNAnexus app/workflow/config-file development at this org is not just a coding
task — it runs through a fixed Jira workflow with defined roles, a modified
GitFlow branching model, mandatory automated + human code review, and
Confluence sign-off documentation before anything reaches production. This
reference covers that surrounding process. For the mechanics of raising the
PR itself, use the `pr-workflow` skill; for the Confluence sign-off page, use
the `confluence-docs` skill (mode `create dev-doc`) — both in this plugin.

Source: internal Confluence "Bioinformatics Development Manual" (draft v4) and
"Documentation guide" (draft v6). Treat specifics (ticket statuses, exact
checklist wording) as subject to change — check the live Confluence pages if
something here seems out of date.

## Roles

Every Epic/Story has three assigned people:

- **Driver**: the bioinformatician doing the implementation work.
- **Navigator**: reviews code/documentation regularly, helps guide decisions.
  Does the first-pass PR review unless unavailable (then escalate to a senior
  bioinformatician).
- **Approver**: usually a Band 8 (or Band 7 in specific delegated cases).
  Final sign-off authority — only the Approver can move a Story from
  `SIGNOFF` → `DEPLOY TO PROD`, and from `FINAL Assurance` → `DONE`.

## Jira hierarchy and ticket types

`Workstream` → `Objective` → `Epic` → `Story` (requires sign-off) / `Task`
(no sign-off) → `Subtask`.

A **Story** = one unit of work needing a single sign-off (e.g. one app
release, one config file, one new reference file). If a change spans two
signable things — e.g. bumping an app *and* generating a new static input
file it needs — that's **two Stories** under one Epic, not one Story with two
parts.

**Naming convention for file/app version Stories**: `item_version`, e.g.
`eggd_athena_v1.7.0` or `clinvar_20230218_b37.vcf.gz`.

### Story status flow

`BACKLOG` → `Selected for dev` → (`DESIGN`, if needed) → `DEVELOPMENT` →
`SIGNOFF` → `DEPLOY TO PROD` (or `Wait to deploy` if blocked on something
external, e.g. a wet-lab change) → `FINAL Assurance` → `DONE` (or
`NOT DEPLOYED` if signed off but decided against; `Ditch` if abandoned
earlier).

Transitions into `DEPLOY TO PROD`/`DONE` are Approver-only gates.

## DNAnexus project naming (`00N_` prefixes)

Org convention for what a DNAnexus project number prefix means:

| Prefix | Purpose | Retention |
|---|---|---|
| `001_` | Reference data (e.g. `001_Reference`) — controlled files, assets, genome builds | Long-term |
| `002_` | Production clinical service runs | Long-term / compliance |
| `003_` | General working/development projects | Auto-archived/deleted over time |
| `004_` | Validation projects whose data must be retained for compliance | Long-term, not auto-archived |

For dev/test work specifically, name the project:

```
003_YYMMDD_<kebab-case-topic>
```

e.g. `003_250114_atlas-cnv-validation`. Underscore separates the
`003_YYMMDD_` prefix from the topic; hyphens (kebab-case) separate words
*within* the topic. Share it with `org-emee_1` at CONTRIBUTE.

- `003_` projects are **auto-archived/deleted over time** — fine for
  throwaway dev/test iteration.
- If the project holds testing/validation evidence that needs to be kept
  long-term for compliance, rename the prefix to `004_` instead (not
  auto-archived/deleted).
- If there's a lot of data and not all of it needs retaining, delete the
  unnecessary parts to control storage cost rather than defaulting to `004_`.
- Reference/controlled files land in `001_Reference` at release time (see
  "Release & deploy" below), not in a `003_`/`004_` project.
- Routine clinical analysis runs live in `002_` projects — never build or
  test new apps there.

## Coding conventions

- Style guides: **PEP 8** for Python, **ShellCheck** for Bash. Recommended
  VS Code extensions: GitLens, ShellCheck, Python, Pylance.
- All code is version-controlled in GitHub under
  [github.com/eastgenomics](https://github.com/eastgenomics).
- Dependencies must be listed with pinned versions in `requirements.txt` or
  `pyproject.toml` — this is checked at code review.
- New app repos should start from the org template:
  [DNAnexus_app_template](https://github.com/eastgenomics/DNAnexus_app_template).
- GitHub repos must have Dependabot security alerts enabled.

## Branching (modified GitFlow)

1. Cut a **release branch** for the version being worked towards.
2. From the release branch, cut a **development/feature branch** — ideally
   named to include the Jira ticket ID (e.g. `DI-3631-single-file-mode`).
3. Only commit on the dev/feature branch. Merge into the release branch when
   ready; the release branch merges into `main` only once fully signed off.

PRs for this dev/feature branch are raised against the **release branch**,
not `main` — otherwise follow the `pr-workflow` skill's GitHub Flow steps
(branch → commit → review-fix loop → push → PR), including its Jira-link
guardrail (every PR body must carry a `Jira: <issue-url>` line so the
GitHub↔Jira integration associates the PR with the Story).

### Commits

- Granular commits — one logical change per commit.
- Reference issues being closed in the message, e.g. `fixes #10`, `closes
  #10` (auto-closes the GitHub issue on merge).

### Pull requests — DNAnexus-specific additions

On top of the `pr-workflow` skill's standard flow:

- **CodeRabbit** (AI review, `coderabbit.ai`) runs automatically on every PR.
  All CodeRabbit comments must be addressed or explicitly refuted before
  requesting human review. Suppress it only when genuinely not needed by
  adding `BunnyNo` to the PR title.
- Human review is done via **Reviewable** (reviewable.io, log in with GitHub)
  rather than the native GitHub review UI — it threads comments per line and
  tracks resolution better for larger diffs.
- Code review **cannot** be done by the PR's author. First choice is the
  assigned Navigator; escalate to a senior bioinformatician if unavailable.
- All review comments must be marked resolved before merging.
- If the PR changes a config file, the reviewer should verify any new file
  IDs referenced correspond to expected, signed-off files.

### App "definition of done" — checked at code review

An app is not mergeable/releasable unless it meets all of:

- **App, not applet** for the production artefact (`dx build --app`, then
  `dx publish`).
- Name **prefixed `eggd_`**.
- `developers` and `authorizedUsers` restricted to `org-emee_1` only.
- Uses an appropriate Ubuntu `runSpec.release` (current standard: `24.04`,
  with `"version": "0"`).
- Region is `aws:eu-central-1`.
- `runSpec.timeoutPolicy` is set (no unbounded jobs).
- **Uses `assetDepends`/`execDepends`** rather than compiling/installing
  tools manually inside the app where an asset already exists.
- Bash apps use at minimum `set -e` (fail fast on non-zero exit).
- Dependencies pinned in `requirements.txt`/`pyproject.toml`.
- GitHub repo has Dependabot enabled.

See `configuration.md` and `app-development.md` for the mechanics of each of
these.

## Testing and documentation (before SIGNOFF)

- All functionality should be tested; depth of testing scales with risk/
  complexity. Unit tests are good practice but 100% coverage is not required.
- Testing results are written up in Confluence using the `confluence-docs`
  skill (mode `create dev-doc`) **before** a Story can move to `SIGNOFF`. The
  signed-off testing document must describe exactly the version of code
  being deployed — not an earlier iteration.
- When a Story transitions to `SIGNOFF`, Jira requires a link to the
  Confluence testing/documentation page for that update.
- Design decisions (if a design stage happened) should be recorded either in
  the Confluence doc or the Jira Epic/Story — reviewers check for this.

## Release & deploy (after SIGNOFF)

Once the Approver signs off and moves the Story to `DEPLOY TO PROD`:

1. Create a GitHub release with the version tag, summarising the changes.
2. If an app: `dx build --app` then `dx publish eggd_x/version`.
3. If a controlled file/workflow: move it into the appropriate location
   under `001_Reference` on DNAnexus; delete the dev-project copy once moved.
4. Fill in the Jira Story's Deployment tab, including prod comments (used to
   generate the automated `#egg-announce` Slack message).
5. Transition the Story to `FINAL Assurance`.
6. The Approver checks all deployment actions were completed correctly, then
   moves the Story to `DONE`.

If a signed-off version ends up not being deployed (e.g. superseded before
release), transition to `NOT DEPLOYED` with the reason recorded first.
