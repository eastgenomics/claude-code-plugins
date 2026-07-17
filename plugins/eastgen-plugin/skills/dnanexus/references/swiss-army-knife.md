# DNAnexus Swiss-army-knife — Ad-hoc Job Reference

`app-swiss-army-knife` runs an arbitrary bash command on a DNAnexus worker.
It is the fastest way to run analysis without building a full app.

## Core pattern

```bash
dx run app-swiss-army-knife \
    --destination "project-xxxx:/output/folder/" \
    -iin="file-aaaa" \           # input file 1
    -iin="file-bbbb" \           # input file 2
    -icmd="<bash commands>" \    # command to run
    --name "job_name" \
    --instance-type mem1_ssd1_v2_x4 \
    -y \                         # don't prompt for confirmation
    --brief                      # print only job ID
```

## Input files — how they arrive

Files are downloaded to `/home/dnanexus/` with their **original DNAnexus filename**.
There are no `$in_000` or `$in_000_path` variables — those do not exist in swiss-army-knife.

```bash
# Files arrive as their original name. Find them by glob:
-icmd='BAM=$(ls *.bam); samtools view -c "$BAM"'
-icmd='PERBASE=$(ls *.per-base.bed.gz); tabix "$PERBASE" -R regions.bed'

# If you need a specific file, pass it as the only -iin of that type
# and glob for it — safer than hardcoding filenames
```

## Output files

Write output to the working directory. All files written there are automatically
uploaded to `--destination` when the job finishes.

## Available tools

| Category | Tools |
|---|---|
| Alignment | `samtools`, `bwa`, `sambamba` |
| Variants | `bcftools`, `vcftools` |
| Intervals | `bedtools`, `tabix`, `bgzip` |
| Assembly | `minimap2` |
| Languages | `python3`, `R`, `java 17`, `perl` |
| Utilities | `jq`, `curl`, `wget`, `pigz`, `parallel` |

## Python packages

Python is present but most scientific packages are NOT pre-installed.
Install at job start:

```bash
-icmd='pip install -q numpy matplotlib pandas 2>&1 | tail -2
python3 myanalysis.py'
```

If pip refuses (externally managed environment error):
```bash
pip install --break-system-packages -q numpy matplotlib pandas
# OR
python3 -m venv /tmp/venv && /tmp/venv/bin/pip install -q numpy
# (then use /tmp/venv/bin/python3 for all python calls)
```

## `--destination` — correct form

```bash
# WRONG — cannot use both --project and --destination
dx run app-swiss-army-knife --project "project-xxxx" --destination "/folder/" ...

# RIGHT — combine project and folder in --destination
dx run app-swiss-army-knife --destination "project-xxxx:/folder/" ...
```

## JOB_CMD escaping

In a double-quoted bash `JOB_CMD` string:
- `$VAR` — expands immediately in the outer script (before job runs)
- `\$VAR` — becomes `$VAR` inside the job (expands when job runs)
- `\$(cmd)` — becomes `$(cmd)` inside the job

```bash
SAMPLE="S0024"
JOB_CMD="set -euo pipefail
# Outer expansion (happens now):
echo 'Sample: ${SAMPLE}'       # → echo 'Sample: S0024'

# Inner expansion (happens in job):
BAM=\$(ls *.bam)               # → BAM=$(ls *.bam)
echo \"bam: \$BAM\"            # → echo \"bam: $BAM\"
"
```

For complex commands, use a non-interpolating heredoc:

```bash
read -r -d '' JOB_CMD << 'JOBEOF' || true
set -euo pipefail
BAM=$(ls *.bam)          # no escaping needed — heredoc is literal
echo "bam: $BAM"
JOBEOF

# Then patch in runtime values via sed if needed:
JOB_CMD="${JOB_CMD/SAMPLE_PLACEHOLDER/${SAMPLE_ID}}"
```

## `read` heredoc with `set -e`

The `read` command exits with code 1 at EOF. Under `set -euo pipefail` this
kills the script silently. Always add `|| true`:

```bash
read -r -d '' PYTHON_SCRIPT << 'EOF' || true
import sys
print(sys.argv)
EOF
```

## BAI index placement

bcftools and samtools require the BAI index to sit alongside the BAM:

```bash
# After downloading BAM and BAI as separate -iin files:
mv "$(ls *.bam.bai 2>/dev/null || ls *.bai)" "$(ls *.bam).bai" 2>/dev/null || true
```

For streaming BAMs via signed URL:
```bash
URL=$(dx make_download_url file-xxxx --duration 3600)
dx download file-xxxx-bai -o /tmp/s.bam.bai -f 2>/dev/null
samtools view -F 3852 -q 20 -X "${URL}" "/tmp/s.bam.bai" 1 2 3 ...
# Pass URL##idx##/path/to.bai for tools that need it inline
```

## Checking job output files

When a job completes, get its output file IDs directly — do not rely on
folder listing if multiple jobs have written the same filename:

```bash
# Get output file IDs from a specific job
dx describe job-xxxx --json | python3 -c "
import sys, json
d = json.load(sys.stdin)
for k, v in d.get('output', {}).items():
    if isinstance(v, list):
        for item in v:
            if isinstance(item, dict):
                print(k, item.get('\$dnanexus_link',''))
    elif isinstance(v, dict):
        print(k, v.get('\$dnanexus_link',''))
"
# Download the specific file
dx download file-xxxx -o output.tsv -f
```

## Polling jobs from a script

```bash
# Poll a single job until terminal
JOB_ID="job-xxxx"
while true; do
    STATE=$(dx describe "$JOB_ID" --json 2>/dev/null | \
            python3 -c "import sys,json; print(json.load(sys.stdin)['state'])")
    case "$STATE" in
        done)    echo "Done"; break ;;
        failed)  echo "Failed"; exit 1 ;;
        *)       echo "State: $STATE"; sleep 60 ;;
    esac
done
```

Note: `sleep 60` between polls will trigger "delegate needs attention" alerts
in the pi agent framework. This is **normal** — the subagent is sleeping,
not stuck. The alert can be safely acknowledged.

## Polling multiple jobs until all terminal

⚠️ The name-based `dx find jobs --name` pattern below only counts jobs from the
*current* batch if the name pattern has never been used before. On a repeat run
with the same prefix, it also counts every historical job — including failed
ones from earlier batches — which inflates `done`/`failed` counts and can
trigger downstream steps prematurely. Prefer the job-ID-tracking-file approach
in `../SKILL.md` → Common Gotchas → "Job tracking" for anything run more than
once with the same name prefix. This form is fine for a one-off batch that
won't be resubmitted under the same name.

```bash
# Find all jobs matching a name pattern and wait for completion
poll_jobs() {
    local NAME_PATTERN="$1"
    local PROJECT="$2"
    while true; do
        RUNNING=$(dx find jobs --project "$PROJECT" \
            --name "$NAME_PATTERN" --state running 2>/dev/null | wc -l)
        RUNNABLE=$(dx find jobs --project "$PROJECT" \
            --name "$NAME_PATTERN" --state runnable 2>/dev/null | wc -l)
        FAILED=$(dx find jobs --project "$PROJECT" \
            --name "$NAME_PATTERN" --state failed 2>/dev/null | wc -l)
        DONE=$(dx find jobs --project "$PROJECT" \
            --name "$NAME_PATTERN" --state done 2>/dev/null | wc -l)
        echo "running=$RUNNING runnable=$RUNNABLE failed=$FAILED done=$DONE"
        [ "$RUNNING" -eq 0 ] && [ "$RUNNABLE" -eq 0 ] && break
        sleep 60
    done
    [ "$FAILED" -gt 0 ] && { echo "ERROR: $FAILED jobs failed"; return 1; }
}

poll_jobs "cobalt_bootstrap_*" "project-xxxx"
```

## Sense-checking job outputs

Always verify key outputs after submitting a test job before submitting the
full cohort:

```bash
# Pattern: submit one test job, verify output, then submit all
bash submit_jobs.sh SAMPLE_S0024   # test one sample

# Download and check output
dx download "project-xxxx:/results/S0024.output.tsv" -o /tmp/check.tsv -f
python3 - << 'EOF'
import pandas as pd, sys
df = pd.read_csv('/tmp/check.tsv', sep='\t')
assert len(df) > 0, "Empty output"
print(f"OK: {len(df)} rows")
EOF

# If check passes, submit all
bash submit_jobs.sh   # no args = all samples
```

## Instance type reference

| Instance | RAM | CPU | Use case |
|---|---|---|---|
| `mem1_ssd1_v2_x2` | 7.5 GB | 2 | tabix, bedtools intersect |
| `mem1_ssd1_v2_x4` | 7.5 GB | 4 | bcftools, Python plotting |
| `mem2_ssd1_v2_x4` | 16 GB | 4 | Java tools (COBALT, AMBER, PURPLE) |
| `mem2_ssd1_v2_x8` | 32 GB | 8 | Multi-sample aggregation |
| `mem3_ssd1_v2_x4` | 30 GB | 4 | Large Java heap requirements |
