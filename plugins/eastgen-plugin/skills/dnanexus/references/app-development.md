# DNAnexus App Development

## Overview

Apps and applets are executable programs that run on the DNAnexus platform. At this organisation, apps use a **Bash entry point** (`src/code.sh`) that handles all DNAnexus I/O, calling a **pure Python CLI** for the core logic. The Python code has no dxpy dependency and can be tested locally without a DNAnexus account.

## Applets vs Apps

- **Applets**: Data objects that live inside a project. Used for development, testing, and internal pipelines.
- **Apps**: Versioned, shareable executables not tied to a project. Published for org-wide use.

Build an applet with `dx build`, promote to an app with `dx build --app`.

**Production requirement**: the org's PR/code-review checklist requires the final deployed artefact to be an **app, not an applet** (`dx build --app`, then `dx publish eggd_x/version`). Applets are fine for `003_`/`004_` development-project testing, but a Story cannot go to `DEPLOY TO PROD` as a bare applet.

## App Directory Structure

```
eggd_myapp/
├── dxapp.json
├── src/
│   └── code.sh                         # Bash entry point
├── resources/
│   ├── home/dnanexus/
│   │   ├── myapp/                      # Python source
│   │   │   ├── myapp.py
│   │   │   └── utils/
│   │   └── packages/                   # Bundled .whl files
│   │       ├── numpy-1.22.2-...whl
│   │       └── pandas-1.4.1-...whl
│   └── usr/local/bin/
│       ├── mark-section                # Structured logging utility
│       ├── mark-success                # Job success marker
│       └── jq-linux64                  # Statically linked jq binary
└── requirements.txt                    # For local dev reference only
```

### The `resources/` overlay

The `resources/` directory is **overlaid onto the execution filesystem at build time**. Files map directly to their path after `resources/`:

| In repo | On worker |
|---|---|
| `resources/home/dnanexus/myapp/myapp.py` | `/home/dnanexus/myapp/myapp.py` |
| `resources/home/dnanexus/packages/*.whl` | `/home/dnanexus/packages/*.whl` |
| `resources/usr/local/bin/mark-section` | `/usr/local/bin/mark-section` |

There is no runtime path resolution needed — files are already on the filesystem when `code.sh` runs. The working directory on startup is `/home/dnanexus`.

## `src/code.sh` — The Bash Entry Point

DNAnexus calls the `main()` function in `code.sh`. Input variables are available as Bash variables matching the `inputSpec` names. Put all executable logic inside `main()` and do not call it explicitly — DNAnexus owns the entry-point lifecycle, and top-level shell code can yield misleading lifecycle/output behaviour.

### Full pattern

```bash
#!/bin/bash
set -exo pipefail

main() {
    echo "Value of vcf: ${vcf}"

    mark-section "Downloading inputs"
    # dx-download-all-inputs downloads all inputs declared in inputSpec
    # to ~/in/<param_name>/<filename>. --parallel speeds up multi-file inputs.
    dx-download-all-inputs --parallel

    # For array inputs, files land in ~/in/<param_name>/. Move if needed:
    find ~/in/vcfs -type f -name "*" -print0 | xargs -0 -I {} mv {} ~/vcfs

    mark-section "Installing packages"
    # Install bundled .whl files in an isolated venv — no network required.
    # On Ubuntu 24.04 workers a plain/sudo pip install can fail under PEP 668;
    # a venv sidesteps the OS-managed environment. See Common Gotchas in SKILL.md.
    python3 -m venv /tmp/myapp-venv
    /tmp/myapp-venv/bin/pip install --no-index --no-deps packages/*
    export PATH="/tmp/myapp-venv/bin:$PATH"

    mark-section "Building arguments"
    # Build argument string for the Python CLI
    args=""
    if [ "$optional_string" ]; then args+="--optional_string ${optional_string} "; fi
    if [ "$flag_param" == true ]; then args+="--flag_param "; fi
    if [ "$int_param" ]; then args+="--int_param ${int_param} "; fi

    mark-section "Running analysis"
    # Call the Python CLI. Pass string args containing spaces separately
    # (quoting in bash breaks when building a string with spaces)
    if [ "$filter_string" ]; then
        python3 myapp/myapp.py --input vcfs/* $args --filter "${filter_string}"
    else
        python3 myapp/myapp.py --input vcfs/* $args
    fi

    mark-section "Uploading outputs"
    # Upload to the worker workspace, wait until the upload finalises, then
    # declare the file in the job output contract. --wait matters when the
    # returned ID is a declared output or consumed by a downstream/child job:
    # it confirms finalisation before the ID is exposed.
    output_file=$(dx upload result.xlsx --wait --brief)
    dx-jobutil-add-output output_file "$output_file" --class=file

    # Upload with structured metadata (queryable via dx describe)
    JSON_DETAILS=$(cat details.json)
    output_file=$(dx upload result.xlsx --wait --brief --details "$JSON_DETAILS")
    dx-jobutil-add-output output_file "$output_file" --class=file

    # Upload multiple files as an array output
    for file in *.vcf.gz; do
        id=$(dx upload "${file}" --wait --brief)
        dx-jobutil-add-output tmp_vcfs "$id" --class=array:file
    done
}
```

### Key Bash helpers

| Command | Purpose |
|---|---|
| `dx-download-all-inputs --parallel` | Downloads all inputSpec files to `~/in/<param>/` |
| `dx-upload-all-outputs --parallel` | Uploads everything under `~/out/<param>/` and populates `job_output.json` |
| `dx-jobutil-add-output <name> <id> --class=<type>` | Sets a job output |
| `dx upload <file> --wait --brief` | Finalises a workspace upload and prints only its file ID |
| `dx upload <file> --wait --brief --details '<json>'` | Finalises an upload with structured metadata |
| `mark-section "label"` | Prints a formatted section header to the job log |
| `mark-success` | Marks the job as successfully completed |

### `dx-download-all-inputs` magic variables

For a real app/applet (this does **not** apply to `app-swiss-army-knife` — see `swiss-army-knife.md`), after `dx-download-all-inputs` the platform sets Bash variables per input field, in addition to placing the file at `~/in/<field>/<filename>`:

| Variable | Value |
|---|---|
| `$<field>_path` | Full path to the downloaded file, e.g. `~/in/vcf/sample.vcf.gz` |
| `$<field>_name` | Filename only (`basename`), e.g. `sample.vcf.gz` |
| `$<field>_prefix` | Filename without its final extension, e.g. `sample.vcf` |

For `array:file` inputs, files land in numbered subfolders (`~/in/<field>/0/`, `~/in/<field>/1/`, …, zero-padded so glob ordering matches input order) — there is no single `$<field>_path` in that case; enumerate with `find`/`ls` as shown below.

### Declared-output contract and downstream consumption

- Every field passed to `dx-jobutil-add-output` must exist in `dxapp.json` `outputSpec` with the exact declared class (`file`, `array:file`, …). DNAnexus rejects undeclared output fields. Keep internal hand-off files internal unless they are deliberately part of the app contract.
- A parent/orchestrator must wait until the child job is terminal `done`, then read the required link from the child's **declared output**. Do not use a transient upload ID as an immediate cross-job transport mechanism.
- Use `--wait` for uploads whose returned ID is declared as output or consumed downstream — it confirms finalisation before the ID is exposed.
- Do not assume a worker token can write to an arbitrary project-qualified destination. Prefer a workspace upload plus `dx-jobutil-add-output`, or write under `~/out/<outputSpec field>/` and use `dx-upload-all-outputs`.

### `dx-upload-all-outputs` convention (counterpart)

As an alternative to manual `dx upload` + `dx-jobutil-add-output` calls, write output files into `~/out/<outputSpec_field_name>/` and call `dx-upload-all-outputs --parallel` once at the end of `code.sh`. Any file placed under `~/out/output_file/` is uploaded and automatically linked to the `output_file` output. This is less explicit than manual upload but avoids repeating `dx-jobutil-add-output` boilerplate for apps with many outputs.

### DNAnexus environment variables

Available in `code.sh` at runtime:

| Variable | Value |
|---|---|
| `$DX_PROJECT_CONTEXT_ID` | The project the job is running in |
| `$DX_JOB_ID` | The ID of the current job |
| `$DX_WORKSPACE_ID` | The job's temporary workspace project |

### Input variable types in Bash

| `dxapp.json` class | Bash variable |
|---|---|
| `string` | `$my_string` — plain string |
| `boolean` | `$my_bool` — literal `"true"` or `"false"` |
| `int` / `float` | `$my_num` — plain number |
| `file` | `$my_file` — JSON link e.g. `{"$dnanexus_link": "file-xxxx"}` |
| `array:file` | `${my_files[@]}` — bash array of JSON links |

After `dx-download-all-inputs`, downloaded files are at `~/in/<param_name>/<filename>`.

## Python CLI — No dxpy

The Python component is a standard CLI tool using `argparse`. It reads and writes local files only.

```python
# resources/home/dnanexus/myapp/myapp.py
import argparse
from pathlib import Path

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--input', nargs='+', required=True)
    parser.add_argument('--output_dir', default='.')
    parser.add_argument('--optional_flag', action='store_true')
    args = parser.parse_args()

    # Process local files, write output to args.output_dir
    # No dxpy imports needed

if __name__ == '__main__':
    main()
```

**Why no dxpy in the Python code?**

- The Python logic can be unit tested locally with no DNAnexus account
- Tests run in CI (GitHub Actions) without platform access
- Keeps concerns separate: DNAnexus I/O in Bash, data logic in Python

## Bundling Python Packages

Pre-download `.whl` files compatible with the execution environment and place them in `resources/home/dnanexus/packages/`.

- Ubuntu **24.04** (current standard): Python 3.12, `cp312`, `manylinux_2_39_x86_64`
- Ubuntu **20.04** (legacy apps only): Python 3.8, `cp38`, `manylinux2014_x86_64`

Install in `code.sh` into an isolated environment:
```bash
python3 -m venv /tmp/myapp-venv
/tmp/myapp-venv/bin/pip install --no-index --no-deps packages/*
export PATH="/tmp/myapp-venv/bin:$PATH"
```

`--no-index` prevents pip from contacting PyPI. `--no-deps` prevents pip from trying to resolve dependencies at install time (all deps must themselves be present as `.whl` files). Ubuntu 24.04 manages its system Python under PEP 668, so install into a virtual environment rather than the worker's system Python (a `sudo pip install` can fail for packages that overlap OS-managed ones).

To find the right wheel files for a package:
```bash
pip download numpy==1.22.2 \
    --platform manylinux2014_x86_64 \
    --python-version 38 \
    --only-binary=:all: \
    -d ./packages/
```

## mark-section

`mark-section` is a utility bundled in `resources/usr/local/bin/` that prints a formatted section header to the job log, making it easier to navigate execution in the DNAnexus monitor. Use it to annotate each logical stage of `code.sh`:

```bash
mark-section "Downloading inputs"
mark-section "Installing packages"
mark-section "Running analysis"
mark-section "Uploading outputs"
```

## Building and Deploying

Build as an applet (into current project):
```bash
dx build eggd_myapp
```

Build as a versioned app (org-wide):
```bash
dx build --app eggd_myapp
```

Publish the built version so authorized users (`org-emee_1`) can run it (unpublished apps are only runnable by developers):
```bash
dx publish eggd_myapp/1.0.0
```

The build process bundles `src/`, `resources/`, and `dxapp.json`, then uploads them to the platform.

### Dynamic child apps

When an app dynamically invokes another executable with `dx run`, test that invocation from the **worker identity**, not your interactive one. A developer can have access to a private app that the parent worker token does not. If worker execution needs a published child app, use the exact reviewed/published version and record its executable ID/version in orchestration provenance. This is a DNAnexus access behaviour, separate from the org policy that a production release must be an approved published app.

## Testing

### Unit tests (local)

Because the Python code has no dxpy dependency, unit tests run locally:

```bash
cd resources/home/dnanexus/myapp
python -m pytest tests/
```

Tests should use local copies of test data files. The organisation's apps use `pytest` with test data stored in `tests/test_data/`.

### GitHub Actions

Add `.github/workflows/pytest.yml` to run tests on pull requests:

```yaml
name: pytest
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with: {python-version: '3.10'}
      - run: pip install -r requirements.txt
      - run: pytest resources/home/dnanexus/myapp/tests/
```

### Platform testing

Run the built applet on the platform with a test input:
```bash
dx run eggd_myapp -iinput_vcf=file-xxxx --watch
```

## Common Issues

**Packages not found**: Ensure `.whl` files match the execution platform — `cp312`/`manylinux_2_39_x86_64` for 24.04 workers, `cp38`/`manylinux2014_x86_64` for legacy 20.04 apps.

**Bash variable quoting**: When building an `$args` string, string arguments containing spaces (e.g. a filter expression) must be passed separately, not via the `$args` string. See the filter pattern in the `code.sh` example above.

**Array inputs**: `dx-download-all-inputs` places each file into `~/in/<param_name>/`. Use `find` or `ls` to enumerate them, then move or use in-place.

**File not in expected location**: Check that the path in `resources/` exactly matches the expected runtime path. `resources/home/dnanexus/foo` → `/home/dnanexus/foo`.
