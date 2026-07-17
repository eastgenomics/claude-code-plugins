# DNAnexus Python SDK (dxpy)

## Overview

The dxpy library provides Python bindings to the DNAnexus API. At this organisation, dxpy is used primarily for **automation scripts and tooling that run outside of DNAnexus jobs** — for example, scripts to search for files, download results, or manage project data.

**Inside apps, dxpy is not used.** App logic lives in `src/code.sh` (which uses the `dx` CLI) and the Python CLI (`resources/home/dnanexus/`) which has no dxpy dependency. See `references/app-development.md` and `references/data-operations.md` for those patterns.

## Installation and Authentication

```bash
pip install dxpy
dx login
dx whoami
dx select project-xxxx   # set default project
```

Via API token:
```python
import dxpy

dxpy.set_security_context({
    "auth_token_type": "Bearer",
    "auth_token": "YOUR_API_TOKEN"
})
dxpy.set_workspace_id("project-xxxx")
```

## Core Classes

### DXFile

```python
import dxpy

# Get handler
file_obj = dxpy.DXFile("file-xxxx")

# Describe
desc = file_obj.describe()
print(desc['name'], desc['size'], desc['state'])

# Download
dxpy.download_dxfile(file_obj.get_id(), "local_file.txt")

# Read contents without saving locally
with file_obj.open_file() as f:
    contents = f.read()

# Update metadata
file_obj.set_properties({"key": "value"})
file_obj.add_tags(["tag1"])
file_obj.rename("new_name.txt")
```

### DXRecord

```python
# Create a metadata record
record = dxpy.new_dxrecord(
    name="sample_metadata",
    types=["SampleMetadata"],
    details={"sample_id": "S001", "tissue": "blood"},
    project="project-xxxx",
    close=True
)

# Read
record = dxpy.DXRecord("record-xxxx")
details = record.get_details()

# Update (must be open)
record.set_details({"processed": True})
record.close()
```

### DXJob

```python
job = dxpy.DXJob("job-xxxx")

# Wait and check state
job.wait_on_done()
desc = job.describe()
print(desc['state'])          # done, failed, etc.
print(desc.get('output', {})) # job outputs

# Terminate
job.terminate()
```

### DXApplet / DXApp

```python
# Run an applet
job = dxpy.DXApplet("applet-xxxx").run({
    "input_file": dxpy.dxlink("file-yyyy"),
    "param": "value"
})

# Run an app by name
job = dxpy.DXApp(name="eggd_generate_variant_workbook").run({
    "vcfs": [dxpy.dxlink("file-yyyy")],
    "summary": "dias"
})
```

### DXWorkflow

```python
analysis = dxpy.DXWorkflow("workflow-xxxx").run({
    "stage-0.vcfs": [dxpy.dxlink("file-yyyy")]
})
analysis.wait_on_done()
```

### DXProject

```python
project = dxpy.DXProject("project-xxxx")
desc = project.describe()

# List folder contents
contents = project.list_folder("/results")
print(contents['objects'])   # list of {id, describe} dicts
print(contents['folders'])   # list of folder paths
```

## High-Level Functions

### File upload/download

```python
# Upload with metadata
file_obj = dxpy.upload_local_file(
    "result.xlsx",
    project="project-xxxx",
    folder="/results",
    name="sample1_report.xlsx",
    properties={"sample": "sample1"},
    tags=["validated"]
)

# Download
dxpy.download_dxfile("file-xxxx", "local_copy.xlsx")
```

### Search functions

```python
# Find files
results = dxpy.find_data_objects(
    classname="file",
    name="*.vcf.gz",
    project="project-xxxx",
    folder="/vcfs",
    describe=True   # include full describe in results
)
for r in results:
    print(r['describe']['name'], r['id'])

# Find by properties
results = dxpy.find_data_objects(
    classname="file",
    properties={"sample": "NA12878"},
    project="project-xxxx"
)

# Find projects
projects = dxpy.find_projects(name="*analysis*", describe=True)

# Find jobs
jobs = dxpy.find_jobs(
    project="project-xxxx",
    state="failed",
    describe=True
)
```

### Links

```python
# Create a dxlink dict from an ID
link = dxpy.dxlink("file-xxxx")
# Returns: {"$dnanexus_link": "file-xxxx"}

# Create with project context
link = dxpy.dxlink("file-xxxx", "project-yyyy")

# Get a job output reference (for chaining jobs)
ref = job.get_output_ref("output_name")
```

## API Methods

For operations not covered by high-level functions:

```python
# Create a folder
dxpy.api.project_new_folder(
    "project-xxxx",
    {"folder": "/results/batch1", "parents": True}
)

# Invite user to project
dxpy.api.project_invite(
    "project-xxxx",
    {"invitee": "user-yyyy", "level": "CONTRIBUTE"}
)

# Close a file
dxpy.api.file_close("file-xxxx")

# Get job logs
log = dxpy.api.job_get_log("job-xxxx")
```

## Error Handling

```python
from dxpy.exceptions import DXAPIError, ResourceNotFound

try:
    file_obj = dxpy.DXFile("file-xxxx")
    desc = file_obj.describe()
except ResourceNotFound:
    print("File not found")
except DXAPIError as e:
    print(f"API error {e.code}: {e}")
```

Common exceptions:
- `DXAPIError` — API request failed
- `ResourceNotFound` — Object doesn't exist
- `PermissionDenied` — Insufficient permissions
- `InvalidInput` — Bad parameter values

## Common Patterns

### Batch processing

```python
files = dxpy.find_data_objects(
    classname="file",
    name="*.vcf.gz",
    project="project-xxxx"
)

jobs = []
for f in files:
    job = dxpy.DXApp(name="eggd_generate_variant_workbook").run({
        "vcfs": [dxpy.dxlink(f["id"])]
    })
    jobs.append(job)

for job in jobs:
    job.wait_on_done()
    print(f"{job.get_id()}: {job.describe()['state']}")
```

### Download all results from a project folder

```python
import os

files = dxpy.find_data_objects(
    classname="file",
    project="project-xxxx",
    folder="/results",
    describe=True
)

os.makedirs("./downloads", exist_ok=True)
for f in files:
    name = f["describe"]["name"]
    dxpy.download_dxfile(f["id"], f"./downloads/{name}")
```

### Check file details metadata

```python
file_obj = dxpy.DXFile("file-xxxx")
desc = file_obj.describe(fields={"name": True, "details": True})
details = desc.get("details", {})
print(details)  # e.g. {"included": 42, "excluded": 105}
```

## Configuration

```python
# Set default project context
dxpy.set_workspace_id("project-xxxx")

# Access current project
print(dxpy.WORKSPACE_ID)
```
