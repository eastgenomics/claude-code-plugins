# Static analysis and sweeps (Step 2, before each review)

Run whatever applies to the files in `git diff --name-status -M <base>...HEAD` and paste the
output, or "no findings", into the review prompt, with each command's exit status and the SHA
it ran at. A missing tool is a failure to report to the
user, never a pass: never write `which X && X || echo`. A required check that fails or cannot run
(the repo's CI checks, and every row that applies, except rows marked optional) blocks Clean and the push, unless the user agrees
to defer it; list it in the prompt as `Deferred to <link>: <check> not run/failing — <why>`.

In the Claude Code sandbox, `uvx` downloads tools from PyPI (`pypi.org`, `files.pythonhosted.org`) through the network proxy. If a download is refused, that is a check that could not run: report it to the user with the refused host — don't skip the row.

| Files | Command |
|---|---|
| Python | `uvx ruff check --select E9,F,B,S,BLE --ignore S101 <files>` — use the repo's own Ruff config if it has one, keeping `F`, `B` and `S` on |
| Python (optional) | `uvx pyright <files>` — if the code is untyped, keep only `reportPossiblyUnbound` / `reportOptionalMemberAccess`-type findings |
| GitHub workflows | `uvx zizmor --offline .github/workflows/` |
| Shell | `shellcheck <files>` for `*.sh` and any file with a `bash`/`sh` shebang (DNAnexus: `src/code.sh`, `resources/usr/bin/*`). For a Terraform `templatefile` script, check the rendered output, not the template |
| Terraform | In each directory with changed `*.tf` files: `terraform fmt -check` and `terraform init -backend=false && terraform validate` |
| Everything | The repo's own CI checks (pytest, bandit, …). After the tests, `git status --porcelain` must be empty |

**Sweeps** — paste the hits:
- `git grep -n` for every identifier, flag, output, path or port the change removes or renames.
- On a version or data-release bump, `git grep -n` for the old version string.
- For each count or "all/always" claim in changed docs, the command that proves it
  (e.g. `set -o pipefail; pytest --collect-only -q | tail -1`).

Never add a suppression (`# nosec`, `# noqa`, `# type: ignore`, `# shellcheck disable=`) without a
reason on the same line.
