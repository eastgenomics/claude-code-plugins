# DNAnexus Job Execution and Workflows

## Overview

A job is created whenever an app or applet runs. It executes on an isolated Linux VM with access to the DNAnexus API, the job's project, and any assets declared in `assetDepends`. New org apps target Ubuntu 24.04 (Python 3.12) — do not assume a worker release from this general description; check the app's `runSpec.release`. Jobs are monitored via the web UI or `dx watch`.

## Running Jobs from the CLI

```bash
# Run an applet with inputs
dx run applet-xxxx -ivcfs=file-yyyy -ifilter="bcftools filter -e 'AF>0.02'" --watch

# Run an app by name
dx run eggd_generate_variant_workbook -ivcfs=file-yyyy -isummary=dias

# Run with instance type override
dx run applet-xxxx -iinput=file-yyyy --instance-type mem2_ssd1_v2_x8

# Run and detach (don't wait)
dx run applet-xxxx -iinput=file-yyyy
```

`--watch` streams job logs to the terminal. Omit to detach immediately after launch.

## Running Jobs via dxpy

```python
import dxpy

# Run an applet
job = dxpy.DXApplet("applet-xxxx").run({
    "vcfs": [dxpy.dxlink("file-yyyy")],
    "filter": "bcftools filter -e 'AF>0.02'"
})

print(f"Job ID: {job.get_id()}")

# Run an app by name
job = dxpy.DXApp(name="eggd_generate_variant_workbook").run({
    "vcfs": [dxpy.dxlink("file-yyyy")],
    "summary": "dias"
})
```

## Monitoring Jobs

```bash
# Stream logs live
dx watch job-xxxx

# Stream full stdout/stderr
dx watch job-xxxx --get-streams

# Check job state
dx describe job-xxxx
```

```python
job = dxpy.DXJob("job-xxxx")

# Wait for completion
job.wait_on_done()

desc = job.describe()
print(f"State: {desc['state']}")  # done, failed, terminated, running, runnable
print(f"Output: {desc.get('output', {})}")
```

## Job States

| State | Meaning |
|---|---|
| `idle` | Created, not yet queued |
| `waiting_on_input` | Waiting for input files to close |
| `runnable` | Queued, waiting for a worker |
| `running` | Executing on a worker |
| `done` | Completed successfully |
| `failed` | Execution failed |
| `terminated` | Manually stopped |

## Getting Job Outputs

```python
job.wait_on_done()
desc = job.describe()
if desc["state"] != "done":
    raise RuntimeError(f"Job did not complete: {desc['state']}")
output = desc["output"]

# For a file output
output_file_id = output["xlsx_report"]["$dnanexus_link"]
dxpy.download_dxfile(output_file_id, "result.xlsx")

# Get output reference for chaining (before job completes)
output_ref = job.get_output_ref("xlsx_report")
```

**Polling rule:** while a job is `runnable` or `running`, `describe` can legitimately return `"output": null` — treat that only as an empty nonterminal value, not a failure. Once the job is `done`, require the expected declared outputs and validate their links/classes before consuming them. For parent/child orchestration, consume a file through the completed child's declared output contract; do not assume a worker upload is project-resolvable before the child finalises it.

## Chaining Jobs

Pass the output of one job as the input to another. DNAnexus resolves the dependency and waits automatically:

```python
qc_job = dxpy.DXApplet("applet-qc").run({"reads": dxpy.dxlink("file-xxxx")})

# Second job will wait for qc_job to complete
align_job = dxpy.DXApplet("applet-align").run({
    "reads": qc_job.get_output_ref("filtered_reads")
})
```

## Tracing Job Provenance

From inside `code.sh`, capture upstream job/workflow information:

```bash
_get_dx_job_ids () {
    job_id=$DX_JOB_ID

    # Get the job ID that created the first input VCF
    vcf=$(awk -F'"' '{print $4}' <<< "${vcfs[0]}")
    analysis_id=$(dx describe --json ${vcf} | jq -r '.createdBy.job')

    if [ "$analysis_id" != "null" ]; then
        # Get the workflow containing that job
        workflow_id=$(dx describe --json "${analysis_id}" | jq -r '.parentAnalysis')

        if [ "$workflow_id" != "null" ]; then
            workflow_name=$(dx describe --json "${workflow_id}" | jq -r '.executableName')
        else
            unset workflow_id
        fi
    fi
}
```

Then pass the provenance to the Python script:
```bash
if [ "$workflow_id" ]; then args+="--workflow ${workflow_name} ${workflow_id} "; fi
if [ "$job_id" ]; then args+="--job_id ${job_id} "; fi
```

## Finding Jobs

```python
# Find failed jobs in a project
jobs = dxpy.find_jobs(
    project="project-xxxx",
    state="failed",
    describe=True
)

for job in jobs:
    print(f"{job['describe']['name']}: {job['id']}")

# Find jobs by tag
jobs = dxpy.find_jobs(
    tags=["batch1"],
    project="project-xxxx"
)
```

```bash
# Find jobs via CLI
dx find jobs --project project-xxxx --state failed
dx find analyses --project project-xxxx
```

## Workflows

Workflows combine multiple apps into a pipeline. Run via the UI or CLI.

```bash
# Run a workflow
dx run workflow-xxxx -istage-0.vcfs=file-yyyy

# Monitor an analysis (workflow run)
dx watch analysis-xxxx
```

```python
# Run a workflow
analysis = dxpy.DXWorkflow("workflow-xxxx").run({
    "stage-0.vcfs": [dxpy.dxlink("file-yyyy")]
})

analysis.wait_on_done()
outputs = analysis.describe()["output"]
```

## Parallel Execution (Subjobs)

For Python entry point apps (rare at this org), subjobs allow parallel processing within a single app:

```python
@dxpy.entry_point('main')
def main(input_files):
    subjobs = []
    for f in input_files:
        subjob = dxpy.new_dxjob(
            fn_input={"file": f},
            fn_name="process_file"
        )
        subjobs.append(subjob)

    return {
        "results": [j.get_output_ref("result") for j in subjobs]
    }

@dxpy.entry_point('process_file')
def process_file(file):
    # process...
    return {"result": output_link}
```

For Bash entry point apps, parallelism is handled by launching separate jobs externally (e.g. from a workflow or an orchestrating script).

## Error Handling

```python
job.wait_on_done()
desc = job.describe()

if desc["state"] == "failed":
    print(f"Job failed: {desc.get('failureReason', 'Unknown')}")
    print(f"Message: {desc.get('failureMessage', '')}")
```

```bash
# Terminate a job
dx terminate job-xxxx
```

## Resource Management

Instance types are set in `dxapp.json` under `regionalOptions`. To override at runtime:

```bash
dx run applet-xxxx -iinput=file-yyyy --instance-type mem3_ssd1_v2_x8
```

```python
job = dxpy.DXApplet("applet-xxxx").run(
    {"input": dxpy.dxlink("file-yyyy")},
    instance_type="mem3_ssd1_v2_x8"
)
```

## Debugging Failed Jobs

A wrapper-reported exit code is a useful summary, not proof of root cause. After **every** failed job:

1. `dx watch job-xxxx --get-streams` — read the complete available worker stdout/stderr, not just the tail.
2. Check `mark-section` output to identify the first application stage that failed.
3. `dx describe job-xxxx --json` — inspect `state`, `failureReason`/`failureMessage`, declared outputs, and inputs.
4. Inspect parent/child jobs or workflow-analysis metadata. Distinguish the **first causal failure** from children that were terminated only because their parent failed or a dependency became unsatisfiable.
5. Record a sanitised causal diagnosis before changing code or resubmitting.

Treat `Exception ignored in: <function DXFile.__del__ ...>` as a log item requiring assessment, not automatically as the root cause — establish whether it occurred before the app entry point, alongside the first command failure, or during cleanup.

For resource issues (OOM, disk full), adjust `regionalOptions` deliberately. For package issues, check that bundled wheels match the configured worker release and Python version (new org apps: Ubuntu 24.04 / Python 3.12; legacy apps: 20.04 / 3.8).
