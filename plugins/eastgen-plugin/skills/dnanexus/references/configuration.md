# DNAnexus App Configuration

## Overview

The `dxapp.json` file defines all metadata, inputs, outputs, and execution requirements for an app or applet.

## Minimal `dxapp.json` for this organisation

```json
{
  "name": "eggd_myapp",
  "title": "eggd_myapp",
  "summary": "One-line description of what the app does",
  "dxapi": "1.0.0",
  "version": "1.0.0",
  "whatsNew": "* v1.0.0 Initial release",
  "authorizedUsers": ["org-emee_1"],
  "developers": ["org-emee_1"],
  "inputSpec": [],
  "outputSpec": [],
  "runSpec": {
    "interpreter": "bash",
    "file": "src/code.sh",
    "distribution": "Ubuntu",
    "release": "24.04",
    "version": "0",
    "timeoutPolicy": {"*": {"hours": 2}}
  },
  "access": {
    "project": "CONTRIBUTE",
    "allProjects": "VIEW",
    "network": ["*"]
  },
  "regionalOptions": {
    "aws:eu-central-1": {
      "systemRequirements": {
        "*": {"instanceType": "mem1_ssd1_v2_x4"}
      }
    }
  }
}
```

## Top-level Metadata Fields

```json
{
  "name": "eggd_myapp",        // App identifier — use eggd_ prefix
  "title": "eggd_myapp",       // Display name (often same as name)
  "summary": "...",            // One-line description shown in UI
  "dxapi": "1.0.0",            // Always "1.0.0"
  "version": "1.2.3",          // Semantic version — required for apps
  "whatsNew": "* v1.2.3 Fixed bug X; * v1.2.0 Added feature Y",
  "authorizedUsers": ["org-emee_1"],   // Who can run the app
  "developers": ["org-emee_1"]         // Who can modify the app
}
```

`whatsNew` is a changelog string. Convention is `* vX.Y.Z description; * vA.B.C description`.

## inputSpec

Each entry defines one input parameter. Inputs are available as Bash variables in `code.sh` matching the `name` field.

```json
{
  "inputSpec": [
    {
      "name": "vcfs",
      "label": "Input VCFs",
      "class": "array:file",
      "optional": false,
      "help": "VEP annotated VCF file(s)"
    },
    {
      "name": "filter",
      "label": "Filter expression",
      "class": "string",
      "optional": true,
      "help": "bcftools filter expression"
    },
    {
      "name": "keep_filtered",
      "label": "Keep filtered variants",
      "class": "boolean",
      "optional": true,
      "default": true,
      "help": "If true, keep filtered variants in a separate sheet"
    },
    {
      "name": "acmg",
      "label": "ACMG sheets",
      "class": "int",
      "optional": true,
      "default": 0,
      "help": "Number of ACMG reporting sheets to add"
    },
    {
      "name": "m_codes",
      "label": "M-codes file",
      "class": "file",
      "optional": true,
      "help": "File containing valid M-codes, one per line"
    }
  ]
}
```

### Input classes

| Class | Bash variable | Notes |
|---|---|---|
| `file` | JSON link string | Downloaded to `~/in/<name>/` by `dx-download-all-inputs` |
| `array:file` | Bash array of JSON links | All files downloaded to `~/in/<name>/` |
| `string` | Plain string | |
| `boolean` | `"true"` or `"false"` | Check with `[ "$var" == true ]` |
| `int` | Integer string | |
| `float` | Float string | |
| `array:string` | Bash array | |

### Input options

| Field | Required | Notes |
|---|---|---|
| `name` | Yes | Must match what `code.sh` references |
| `class` | Yes | See table above |
| `optional` | No | Defaults to `false` |
| `default` | No | Only for optional params |
| `label` | No | Display name in UI |
| `help` | No | Description shown in UI |
| `group` | No | Groups parameters in UI (`"app"`, `"generate_workbook.py"`, etc.) |
| `patterns` | No | File name patterns for file inputs (e.g. `["*.vcf", "*.vcf.gz"]`) |

## outputSpec

```json
{
  "outputSpec": [
    {
      "name": "xlsx_report",
      "label": "Excel workbook",
      "class": "file",
      "patterns": ["*.xlsx"],
      "help": "Output Excel workbook"
    },
    {
      "name": "tmp_vcfs",
      "label": "Intermediate VCFs",
      "class": "array:file",
      "optional": true,
      "help": "Intermediate split/filtered VCF files"
    }
  ]
}
```

Outputs are set in `code.sh` using `dx-jobutil-add-output`.

## runSpec

```json
{
  "runSpec": {
    "interpreter": "bash",
    "file": "src/code.sh",
    "distribution": "Ubuntu",
    "release": "24.04",        // Current standard (requires "version": "0"). Older apps still on 20.04 are legacy, not a template to copy.
    "version": "0",
    "timeoutPolicy": {
      "*": {"hours": 2}        // "*" applies to all entry points
    },
    "assetDepends": [          // See assetDepends section below
      {
        "name": "htslib",
        "project": "project-Fkb6Gkj433GVVvj73J7x8KbV",
        "folder": "/app_assets/htslib/htslib_v1.15.0",
        "version": "1.15.0"
      }
    ]
  }
}
```

`interpreter` is `"bash"` for the standard pattern. `"python3"` is also valid if using a Python entry point directly (rare at this org).

## assetDepends

Assets are pre-built dependency bundles shared across apps. The standard way to include bioinformatics tools like `bcftools`/`htslib` is via `assetDepends` — **not** `execDepends`.

```json
"assetDepends": [
  {
    "name": "htslib",
    "project": "project-Fkb6Gkj433GVVvj73J7x8KbV",
    "folder": "/app_assets/htslib/htslib_v1.15.0",
    "version": "1.15.0"
  }
]
```

Fields:
- `name` — asset name
- `project` — project ID containing the asset record
- `folder` — folder path to the asset record within that project
- `version` — asset version

This is different from the `{"id": {"$dnanexus_link": "record-xxxx"}}` format shown in some documentation — use the `name/project/folder/version` format above.

## access

```json
{
  "access": {
    "project": "CONTRIBUTE",    // Write outputs to the job's project
    "allProjects": "VIEW",      // Read input files from other projects
    "network": ["*"]            // Full internet access (for pip, APIs, etc.)
  }
}
```

`allProjects: "VIEW"` is required when input files may come from a different project than the one the job runs in (common in pipeline workflows). Remove if not needed.

`network: ["*"]` allows outbound internet. Restrict to specific domains if internet is not needed, or omit the `network` key entirely to block network access.

## regionalOptions

Specifies instance types per region. The organisation runs in `aws:eu-central-1`.

```json
{
  "regionalOptions": {
    "aws:eu-central-1": {
      "systemRequirements": {
        "*": {"instanceType": "mem1_ssd1_v2_x4"}
      }
    }
  }
}
```

`"*"` applies to all entry points. Use the entry point function name instead to set per-entry-point instance types.

### Common instance types

| Type | Cores | RAM |
|---|---|---|
| `mem1_ssd1_v2_x2` | 2 | 3.9 GB |
| `mem1_ssd1_v2_x4` | 4 | 7.8 GB — standard default |
| `mem1_ssd1_v2_x8` | 8 | 15.6 GB |
| `mem2_ssd1_v2_x4` | 4 | 15.6 GB |
| `mem2_ssd1_v2_x8` | 8 | 31.2 GB |
| `mem3_ssd1_v2_x8` | 8 | 62.5 GB |
| `mem3_ssd1_v2_x16` | 16 | 125 GB |

Start with `mem1_ssd1_v2_x4`. Scale up only if jobs run out of memory.

## execDepends

Install Ubuntu packages at runtime (less preferred than `assetDepends` for bioinformatics tools):

```json
{
  "runSpec": {
    "execDepends": [
      {"name": "samtools"},
      {"name": "python3-pip"}
    ]
  }
}
```

Packages are installed with `apt-get` from Ubuntu repos. Prefer `assetDepends` for versioned bioinformatics tools.

**Code review requirement**: the org's app checklist explicitly calls out "uses assets over manually compiling in-app" — `assetDepends` (or `execDepends`) should be the default, with the offline `.whl`/venv approaches below reserved for cases where no asset exists yet.

## Python package dependencies

**Do not** install Python packages from PyPI at runtime on a live network by default (`access.network` should be scoped, not left wide open, unless the app genuinely needs it). Instead, bundle `.whl` files in `resources/home/dnanexus/packages/` and install offline in `code.sh`:

```bash
sudo -H python3 -m pip install --no-index --no-deps packages/*
```

Wheel compatibility depends on the worker's Ubuntu release:

| `runSpec.release` | Python | Wheel tag |
|---|---|---|
| `24.04` (current standard) | 3.12 | `cp312`, `manylinux_2_39_x86_64` (or `manylinux2014` for pure-C wheels built long ago) |
| `20.04` (legacy apps only) | 3.8 | `cp38`, `manylinux2014_x86_64` / `manylinux_2_17_x86_64` |

On Ubuntu 24.04 workers, PEP 668 ("externally managed environment") means a plain `pip install` — even offline — can fail for packages that overlap OS-managed ones. If `--break-system-packages` isn't sufficient, install into a venv instead (`python3 -m venv /tmp/env && /tmp/env/bin/pip install --no-index --no-deps packages/*`).

## Timeout policy

```json
"timeoutPolicy": {
  "*": {"hours": 2}
}
```

Set based on expected runtime. Can specify `days`, `hours`, and `minutes` together. Applies to all entry points with `"*"`, or per-function by name.

## Complete example

See `SKILL.md` for the full minimal `dxapp.json`. A realistic production example from this org (note: this app predates the 24.04 migration and still runs on 20.04 — new apps should target 24.04, see the release table above):

```json
{
  "name": "eggd_generate_variant_workbook",
  "title": "eggd_generate_variant_workbook",
  "summary": "Create Excel workbook from VEP annotated vcf",
  "dxapi": "1.0.0",
  "version": "2.11.1",
  "whatsNew": "* v2.11.1 merge hotfix; * v2.11.0 Make Uranus/MYE workbooks compatible for clinvar submission",
  "authorizedUsers": ["org-emee_1"],
  "developers": ["org-emee_1"],
  "inputSpec": [...],
  "outputSpec": [...],
  "runSpec": {
    "interpreter": "bash",
    "file": "src/code.sh",
    "distribution": "Ubuntu",
    "release": "20.04",
    "version": "0",
    "timeoutPolicy": {"*": {"hours": 2}},
    "assetDepends": [
      {
        "name": "htslib",
        "project": "project-Fkb6Gkj433GVVvj73J7x8KbV",
        "folder": "/app_assets/htslib/htslib_v1.15.0",
        "version": "1.15.0"
      }
    ]
  },
  "access": {
    "project": "CONTRIBUTE",
    "allProjects": "VIEW",
    "network": ["*"]
  },
  "regionalOptions": {
    "aws:eu-central-1": {
      "systemRequirements": {
        "*": {"instanceType": "mem1_ssd1_v2_x4"}
      }
    }
  }
}
```
