# DNAnexus Data Operations

## Overview

Inside a DNAnexus app (`code.sh`), data operations are done with `dx` CLI commands and platform-provided Bash helpers. For automation scripts running outside of apps, the dxpy Python SDK is used. This page covers both, with CLI patterns first as they are more relevant to app development.

## Inside an App (Bash CLI)

### Downloading inputs: `dx-download-all-inputs`

The standard way to download all inputs declared in `inputSpec`:

```bash
dx-download-all-inputs --parallel
```

This downloads every input file to `~/in/<param_name>/<original_filename>`. For example, an input named `vcfs` containing `sample.vcf.gz` downloads to `~/in/vcfs/sample.vcf.gz`.

`--parallel` downloads all files concurrently — always use it.

After downloading, move files into a working directory if needed:
```bash
# Move all vcf inputs to a flat working directory
mkdir vcfs
find ~/in/vcfs -type f -print0 | xargs -0 -I {} mv {} ~/vcfs/

# Or reference in-place
python3 myapp/myapp.py --input ~/in/vcfs/*
```

For a single `file` input named `m_codes`:
```bash
# File lands at ~/in/m_codes/<filename>
python3 myapp/myapp.py --m_codes in/m_codes/*
```

### Uploading outputs: `dx upload`

```bash
# Upload a file, get back its ID
output_id=$(dx upload result.xlsx --brief)

# Upload with structured JSON metadata (queryable via dx describe)
JSON_DETAILS='{"included": 42, "excluded": 105, "clinical_indication": "R208"}'
output_id=$(dx upload result.xlsx --brief --details "$JSON_DETAILS")

# Upload from a variable containing JSON
JSON_DETAILS=$(cat details.json)
output_id=$(dx upload result.xlsx --brief --details "$JSON_DETAILS")
```

`--brief` prints only the file ID (`file-xxxx`) — required when capturing the ID in a variable.

### Setting job outputs: `dx-jobutil-add-output`

After uploading, register the file as a job output matching an `outputSpec` entry:

```bash
# Single file output
output_id=$(dx upload result.xlsx --brief)
dx-jobutil-add-output xlsx_report "$output_id" --class=file

# Array file output (call once per file)
for file in *.vcf.gz; do
    id=$(dx upload "${file}" --brief)
    dx-jobutil-add-output tmp_vcfs "$id" --class=array:file
done
```

The first argument must match the `name` in `outputSpec`. `--class` must match the `class` in `outputSpec`.

### Finding existing files: `dx find data`

Search for files in a project, useful for deduplication or provenance checks:

```bash
project_id=$DX_PROJECT_CONTEXT_ID

# Find files matching a name pattern in a project
matching=$(dx find data --path "${project_id}":/ --name "sample_*" --brief | wc -l)

# Check if an output with a given prefix already exists
matches=$(dx find data --path "${project_id}":/ --name "${output_prefix}*" --brief | wc -l)
if [ $matches -ne 0 ]; then
    echo "Output already exists, incrementing version"
fi
```

`--brief` prints only object IDs, one per line — use `wc -l` for counting.

### Inspecting objects: `dx describe`

Retrieve JSON metadata for any platform object:

```bash
# Describe a file
dx describe --json file-xxxx

# Get the job that created a file (for provenance)
vcf_id=$(awk -F'"' '{print $4}' <<< "${vcfs[0]}")  # extract ID from dxlink JSON
creating_job=$(dx describe --json ${vcf_id} | jq -r '.createdBy.job')

# Get the workflow containing that job
workflow_id=$(dx describe --json "${creating_job}" | jq -r '.parentAnalysis')
workflow_name=$(dx describe --json "${workflow_id}" | jq -r '.executableName')
```

`jq` is available as `jq-linux64` (bundled in `resources/usr/local/bin/`, symlinked or named `jq` on the path). Use `jq -r` for raw string output (no quotes).

### Environment variables

Available in `code.sh` during job execution:

| Variable | Description |
|---|---|
| `$DX_PROJECT_CONTEXT_ID` | Project ID the job is running in (e.g. `project-xxxx`) |
| `$DX_JOB_ID` | ID of the current job (`job-xxxx`) |
| `$DX_WORKSPACE_ID` | Temporary workspace project for the job |
| `$DX_RESOURCES_ID` | Asset resources project |

### Extracting a file ID from a dxlink input variable

File inputs arrive as JSON strings: `{"$dnanexus_link": "file-xxxx"}`. Extract the ID:

```bash
# For a single file input named 'vcf'
file_id=$(awk -F'"' '{print $4}' <<< "${vcf}")

# For array inputs, iterate
for link in "${vcfs[@]}"; do
    file_id=$(awk -F'"' '{print $4}' <<< "${link}")
    echo "File: $file_id"
done
```

### Full provenance pattern (from `code.sh`)

Capture upstream job and workflow IDs to store as metadata:

```bash
_get_dx_job_ids () {
    job_id=$DX_JOB_ID

    vcf=$(awk -F'"' '{print $4}' <<< "${vcfs[0]}")
    analysis_id=$(dx describe --json ${vcf} | jq -r '.createdBy.job')

    if [ "$analysis_id" != "null" ]; then
        workflow_id=$(dx describe --json "${analysis_id}" | jq -r '.parentAnalysis')
        if [ "$workflow_id" != "null" ]; then
            workflow_name=$(dx describe --json "${workflow_id}" | jq -r '.executableName')
        else
            unset workflow_id
        fi
    fi
}
```

---

## Outside an App (dxpy Python SDK)

For automation scripts, data pipelines, or tooling that runs outside of a DNAnexus job, use the dxpy Python SDK.

### Authentication

```bash
dx login          # interactive
dx select <project>
```

Or via token:
```python
import dxpy
dxpy.set_security_context({
    "auth_token_type": "Bearer",
    "auth_token": "YOUR_TOKEN"
})
```

### Uploading files

```python
import dxpy

file_obj = dxpy.upload_local_file(
    "result.xlsx",
    project="project-xxxx",
    folder="/results",
    name="sample1_report.xlsx",
    properties={"sample": "sample1", "pipeline": "dias"},
    tags=["validated"]
)
print(file_obj.get_id())
```

### Downloading files

```python
dxpy.download_dxfile("file-xxxx", "local_output.xlsx")

# Or via handler
file_obj = dxpy.DXFile("file-xxxx")
dxpy.download_dxfile(file_obj.get_id(), "local_output.xlsx")
```

### Searching for files

```python
# Find files by name pattern
results = dxpy.find_data_objects(
    classname="file",
    name="*.vcf.gz",
    project="project-xxxx",
    folder="/vcfs",
    describe=True
)

for result in results:
    print(f"{result['describe']['name']}: {result['id']}")

# Find by properties
results = dxpy.find_data_objects(
    classname="file",
    properties={"sample": "NA12878"},
    project="project-xxxx"
)
```

### File metadata

```python
file_obj = dxpy.DXFile("file-xxxx")
desc = file_obj.describe()
print(desc['name'], desc['size'], desc['details'])

# Update properties
file_obj.set_properties({"processed": "true"})
file_obj.add_tags(["reviewed"])
```

### Moving and organising

```python
# Create folder
dxpy.api.project_new_folder(
    "project-xxxx",
    {"folder": "/results/batch1", "parents": True}
)

# Move file
dxpy.DXFile("file-xxxx", project="project-xxxx").move("/results/batch1")

# Clone to another project
dxpy.DXFile("file-xxxx").clone("project-yyyy", folder="/imported")
```

### Batch download

```python
files = dxpy.find_data_objects(
    classname="file",
    project="project-xxxx",
    folder="/results"
)

for f in files:
    obj = dxpy.DXFile(f['id'])
    name = obj.describe()['name']
    dxpy.download_dxfile(f['id'], f"./downloads/{name}")
```

## File Details Metadata

DNAnexus files support a `details` field — structured JSON attached to the file object, queryable via `dx describe`. This org uses it to store variant counts and clinical information on output reports:

```bash
# In code.sh — attach details at upload time
JSON_DETAILS=$(cat details.json)
output_id=$(dx upload report.xlsx --brief --details "$JSON_DETAILS")
```

```python
# In a script — read details from an existing file
file_obj = dxpy.DXFile("file-xxxx")
details = file_obj.describe(fields={"details": True}).get("details", {})
print(details)  # e.g. {"included": 42, "excluded": 105, "clinical_indication": "R208"}
```

---

## Chromosome Naming — Critical Gotcha

BAMs, BED files, VCFs, and reference FASTAs must all use the **same chromosome
naming convention**. A mismatch produces **silent empty output** — no error is raised.

- **no-chr**: `1`, `2`, ..., `22`, `X`, `Y`  (e.g. Sentieon BWA-MEM2 at this org)
- **chr-prefixed**: `chr1`, `chr2`, ..., `chrX`, `chrY`

```bash
# Check BAM chromosome naming
samtools view -H sample.bam | grep "^@SQ" | head -3

# Check BED file naming
head -1 regions.bed | cut -f1

# Check VCF
bcftools view -h sample.vcf.gz | grep "^#CHROM" | head -1

# Strip chr prefix from BED to match no-chr BAMs
awk '{gsub(/^chr/,"",$1); print}' chr_prefixed.bed > nochr.bed

# Add chr prefix to BED to match chr-prefixed BAMs
awk '{if($1!~/^#/) $1="chr"$1; print}' OFS="\t" nochr.bed > chr_prefixed.bed
```

## Multiple Versions of Same-Named Files

When multiple jobs write to the same DNAnexus folder with the same filename,
all versions coexist (with different file IDs). `dx download -r` may retrieve
an older version.

To download the output from a **specific job**:

```bash
# Get the file ID from a job's output
FID=$(dx describe job-xxxx --json 2>/dev/null | python3 -c "
import sys, json
d = json.load(sys.stdin)
for k, v in d.get('output', {}).items():
    if isinstance(v, list):
        for item in v:
            fid = item.get('\$dnanexus_link', '')
            if fid: print(fid)
    elif isinstance(v, dict):
        fid = v.get('\$dnanexus_link', '')
        if fid: print(fid)
" | head -1)
dx download "$FID" -o specific_output.tsv -f
```
