# Static analysis and sweeps (Step 2, before each review)

Run whatever applies to the files in `git diff --name-status -M <base>...HEAD` and paste the
output, or "no findings", into the review prompt, with each command's exit status and the SHA
it ran at. A missing tool is a failure to report to the
user, never a pass: never write `which X && X || echo`. A required check that fails or cannot run
(the repo's CI checks, and every row that applies, except rows marked optional) blocks Clean and the push, unless the user agrees
to defer it; list it in the prompt as `Deferred to <link>: <check> not run/failing — <why>`.

**In the Claude Code sandbox:**
- `uvx` and `pip` can't write to `~/.cache` or `~/.local/share`, so they fail with "Read-only file system". Point their caches at `$TMPDIR` in **the same Bash call** as the tool (environment variables don't carry over between calls):
  ```bash
  export UV_CACHE_DIR="$TMPDIR/uv-cache" UV_TOOL_DIR="$TMPDIR/uv-tools" UV_PYTHON_INSTALL_DIR="$TMPDIR/uv-python" PIP_CACHE_DIR="$TMPDIR/pip-cache"; uvx ruff check …
  ```
- They download from PyPI (`pypi.org`, `files.pythonhosted.org`) through the network proxy. If a download is refused, that is a check that could not run: report it to the user with the refused host. Don't skip the row.
- Don't pipe a check into `tail`/`head` when recording its exit status: the pipe reports the status of `tail`, not of the check. Use `set -o pipefail`, or write the output to a file under `$TMPDIR` and read it.
- **Checks the sandbox can't start** (e.g. `terraform validate`/`terraform test`, where the provider plugin fails to start). A green CI run at the **same SHA** counts as evidence for that check only, never for a check that ran here and failed. Paste the evidence into the prompt as `CI evidence at <sha>`, not as a reason not to look:
  - find the run with `gh run list --commit <sha> --json databaseId,workflowName,event,conclusion`;
  - `gh run view <run-id> --json headSha,conclusion,event` with `headSha` equal to the reviewed SHA;
  - the **step** that runs the check, with conclusion `success`: `gh run view <run-id> --json jobs --jq '.jobs[] | .name as $j | .steps[] | "\($j) / \(.name): \(.conclusion)"'`. A job's `success` alone is not enough. A `skipped` job or step (e.g. path-filtered out) is no evidence. A step under `continue-on-error` also reports `success` when it fails, so check the workflow file at that SHA (`git show <sha>:.github/workflows/<file>`). If `continue-on-error` is set on that step or job, the run is no evidence.
  Each is a bare `gh` call. With no CI run at that exact SHA (e.g. on local fix commits), the check could not run: tell the user, or defer it as above.

| Files | Command |
|---|---|
| Python | `uvx ruff check --select E9,F,B,S,BLE --ignore S101 <files>` — use the repo's own Ruff config if it has one, keeping `F`, `B` and `S` on |
| Python (optional) | `uvx pyright <files>` — if the code is untyped, keep only `reportPossiblyUnbound` / `reportOptionalMemberAccess`-type findings |
| GitHub workflows | `uvx zizmor --offline .github/workflows/` |
| Shell | `shellcheck <files>` for `*.sh` and any file with a `bash`/`sh` shebang (DNAnexus: `src/code.sh`, `resources/usr/bin/*`). For a Terraform `templatefile` script, check the rendered output, not the template |
| Terraform | In each module with changed `*.tf`, `*.tftest.hcl` or `*.tftest.json` files (a test file under `tests/` belongs to its parent module): `terraform fmt -check` and `terraform init -backend=false && terraform validate`. Add `terraform test` if the module has test files (`*.tftest.hcl` or `*.tftest.json`, in the module root or `tests/`). `terraform test` **applies real infrastructure by default**, so run it only if every `run` block uses `command = plan` or mocked providers (`mock_provider`, `override_resource`). Otherwise don't run it from the review: use CI evidence at the same SHA, or ask the user. If the sandbox can't start the provider, use CI evidence at the same SHA (above) |
| IaC (Terraform, CloudFormation, Kubernetes, Dockerfile) — required if the repo's CI runs checkov, optional otherwise | If the repo hash-locks checkov (e.g. `ci/requirements-checkov.txt`), install from that file into a venv: `python3 -m venv "$TMPDIR/checkov-venv" && "$TMPDIR/checkov-venv/bin/pip" install --no-cache-dir --require-hashes -r <lockfile> && "$TMPDIR/checkov-venv/bin/checkov" -d <dir> <the CI job's flags>`. Don't use `uvx --with-requirements`: it resolves checkov separately and breaks on a dependency mismatch with the lockfile. Without a lockfile, use `uvx checkov -d <dir> <the CI job's flags>` (e.g. `--external-checks-dir`, skip lists), so the same checks run |
| Everything | The repo's own CI checks (pytest, bandit, …). After the tests, `git status --porcelain` must be empty |

**Sweeps** — paste the hits:
- `git grep -n` for every identifier, flag, output, path or port the change removes or renames.
- On a version or data-release bump, `git grep -n` for the old version string.
- For each count or "all/always" claim in changed docs, the command that proves it
  (e.g. `set -o pipefail; pytest --collect-only -q | tail -1`).

Never add a suppression (`# nosec`, `# noqa`, `# type: ignore`, `# shellcheck disable=`) without a
reason on the same line.
