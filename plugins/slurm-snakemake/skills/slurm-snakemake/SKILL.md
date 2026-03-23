---
name: slurm-snakemake
description: Snakemake workflows on Slurm HPC clusters. Use when writing Snakefiles, configuring Slurm executor profiles, debugging failed jobs, setting resource directives, or running pipelines on a cluster. Covers Snakemake 8+ executor plugin system.
---

# Snakemake + Slurm (Executor Plugin, v8+)

Snakemake 8 replaced `--cluster`/`--cluster-config` with **executor plugins**. The old Snakemake-Profiles/slurm cookiecutter approach is obsolete.

```bash
pip install snakemake-executor-plugin-slurm
```

---

## Running Workflows on Slurm

```bash
# Basic execution
snakemake --executor slurm --jobs 100

# With a profile (recommended)
snakemake --profile slurm/

# With workflow-local profile
snakemake --workflow-profile slurm/

# Run specific target
snakemake --executor slurm --jobs 100 --until all_preprocess

# Rerun incomplete jobs
snakemake --executor slurm --rerun-incomplete
```

**Run Snakemake on the head node inside tmux** — the scheduler process must stay alive to monitor jobs and clean up `temp()` files:
```bash
tmux new-session -s pipeline
snakemake --profile slurm/ 2>&1 | tee logs/run-$(date +%Y%m%d_%H%M%S).log
```

---

## Profile Configuration

Profiles are YAML files where every CLI flag becomes a key. Store in a project-local directory (e.g., `slurm/config.yaml`) or globally at `~/.config/snakemake/<name>/config.yaml`.

```yaml
# slurm/config.yaml
executor: slurm
jobs: 100
latency-wait: 60
keep-going: true
rerun-triggers: mtime
use-conda: true

default-resources:
    slurm_partition: YOUR_PARTITION
    slurm_account: YOUR_ACCOUNT
    mem_mb: 4000

# Per-rule resource overrides
set-threads:
    align: 8
    sort: 4

set-resources:
    align:
        mem_mb: 32000
    heavy_merge:
        mem_mb: 500000
        slurm_partition: highmem

# What's on shared filesystem (most HPC setups: everything)
shared-fs-usage:
    - persistence
    - software-deployment
    - sources
    - source-cache
```

> **Caveat: `runtime` in profiles.** Setting `runtime` via `default-resources` or `set-resources` in the profile may not propagate to Slurm's `--time` flag (verified on Snakemake 9.x / executor plugin 2.5.x — all jobs received the 1-minute default regardless of profile values). **Set `runtime` directly in rule `resources:` blocks** where it reliably maps to `--time`. `mem_mb` and `set-threads` work correctly from profiles.

### Automatic Partition Selection

On clusters with numeric partition names (e.g., `20`, `24`), prefer automatic partition selection over setting `slurm_partition` directly. This avoids a known bug where numeric partition names are summed in group jobs (see [Grouping Jobs](#grouping-jobs)).

```yaml
# partitions.yaml — define available partitions and their limits
partitions:
  "20":
    max_runtime: "21-00:00:00"
    max_mem_mb: 245760
    max_cpus_per_task: 32
    max_nodes: 1
  "24":
    max_runtime: "21-00:00:00"
    max_mem_mb: 245760
    max_cpus_per_task: 48
    max_nodes: 1
```

```yaml
# slurm/config.yaml — point to partition config instead of setting slurm_partition
executor: slurm
slurm-partition-config: partitions.yaml
default-resources:
    mem_mb: 4000
```

The plugin selects the best-fit partition based on each job's resource requirements. See the [Slurm executor plugin docs](https://snakemake.github.io/snakemake-plugin-catalog/plugins/executor/slurm.html#automatic-partition-selection) for details.

**Usage:**
```bash
snakemake --workflow-profile slurm/
# Or globally:
export SNAKEMAKE_PROFILE=myprofile
```

---

## Writing Rules with Resources

### Basic Rule

```python
rule align:
    input:
        reads="data/{sample}.fastq.gz",
        index="ref/genome.fa.fai"
    output:
        "results/{sample}.bam"
    threads: 8
    resources:
        mem_mb=32000,
        runtime=120,
    log:
        "logs/align/{sample}.log"
    shell:
        "bwa mem -t {threads} ref/genome.fa {input.reads} 2> {log} "
        "| samtools sort -@ {threads} -o {output}"
```

### Resource Directives Reference

| Snakemake resource | Slurm flag | Notes |
|---|---|---|
| `threads` | `--cpus-per-task` | Set via `threads:`, not `resources:` |
| `mem_mb` | `--mem` | Total memory in MB |
| `mem_mb_per_cpu` | `--mem-per-cpu` | **Mutually exclusive** with `mem_mb` |
| `runtime` | `--time` | Minutes |
| `slurm_partition` | `--partition` | Queue name |
| `slurm_account` | `--account` | Billing account |
| `slurm_extra` | *(appended raw)* | Arbitrary sbatch flags |
| `constraint` | `--constraint` | Node features |
| `gpu` | `--gpus` | GPU count |

### GPU Rules

```python
rule train_model:
    input: "data/features.h5"
    output: "models/trained.pt"
    threads: 4
    resources:
        mem_mb=64000,
        runtime=480,
        slurm_partition="gpu",
        gpu=1,
        slurm_extra="'--gres=gpu:a100:1'"
    shell:
        "python train.py --input {input} --output {output}"
```

### Dynamic Resources with `attempt`

Scale resources on retry to handle OOM kills:

```python
rule memory_hungry:
    input: "data/{sample}.h5"
    output: "results/{sample}.parquet"
    threads: 4
    resources:
        mem_mb=lambda wildcards, attempt: 32000 * attempt,
        runtime=lambda wildcards, attempt: 120 * attempt,
    retries: 3
    shell:
        "python process.py {input} {output}"
```

Run with `--retries 3` or set in profile. First attempt gets 32 GB, second 64 GB, third 96 GB.

### Input-Dependent Resources

```python
rule sort:
    input: "data/{sample}.bam"
    output: "sorted/{sample}.bam"
    threads: 4
    resources:
        mem_mb=lambda wildcards, input: max(int(input.size_mb * 2.5), 4000),
        runtime=lambda wildcards, input: max(int(input.size_mb / 100), 30),
    shell:
        "samtools sort -@ {threads} -m 2G -o {output} {input}"
```

---

## Keep Snakefiles Portable

**Do not hardcode cluster-specific values in rules.** Put partitions, accounts, and cluster-specific resources in profiles so the same workflow runs on different clusters.

```python
# BAD - hardcoded to one cluster
rule align:
    resources:
        slurm_partition="our-lab-queue",
        slurm_account="lab123"

# GOOD - defaults come from profile
rule align:
    resources:
        mem_mb=32000,
        runtime=120
```

---

## Intermediate File Cleanup

### `temp()` — Automatic Deletion

Mark intermediate files with `temp()` so Snakemake deletes them once all downstream consumers finish. Critical on shared HPC storage where intermediate files (unsorted BAMs, raw extracts) can consume terabytes.

```python
rule align:
    output: temp("intermediates/{sample}.unsorted.bam")
    shell: "bwa mem ... | samtools view -b > {output}"

rule sort:
    input: "intermediates/{sample}.unsorted.bam"
    output: "results/{sample}.sorted.bam"
    shell: "samtools sort {input} -o {output}"
# After sort completes, Snakemake deletes the unsorted BAM automatically
```

Deletion is performed by the **Snakemake scheduler on the head node**, not by the Slurm job itself. If the scheduler process dies (e.g., tmux session lost), cleanup stops and temp files remain. This is why running in tmux is critical — not just for job monitoring, but for cleanup.

Use `--notemp` to keep all temp files for debugging. Use `--delete-temp-output` to retroactively delete temp-marked files from a completed run.

### `protected()` — Write-Protect Expensive Outputs

Mark outputs that are expensive to reproduce. Snakemake sets them read-only (chmod 444) after creation, preventing accidental overwrite from `--forceall` or re-runs.

```python
rule expensive_alignment:
    output: protected("results/{sample}.final.bam")
    shell: "..."
```

---

## Shadow Rules

Run rules in isolated temporary directories to contain undeclared temp files. The shadow directory is created under `.snakemake/shadow/`, inputs are symlinked in, the rule executes there, declared outputs are moved back, and the shadow directory is deleted.

```python
rule messy_tool:
    input: "data/{sample}.fastq"
    output: "results/{sample}.counts.txt"
    shadow: "minimal"
    shell:
        """
        # This tool dumps temp files everywhere — shadow contains the mess
        run_messy_tool --input {input} --tmp-prefix tmp_
        mv tmp_counts.txt {output}
        """
```

| Shadow mode | What's available in shadow dir | Use case |
|---|---|---|
| `"minimal"` | Only symlinks to declared inputs | Cleanest isolation |
| `"shallow"` | Symlinks to all top-level files/dirs | Tool needs config files, scripts |
| `"full"` | Recursive symlinks of entire project tree | Tool navigates relative paths |
| `"copy-minimal"` | Copies (not symlinks) of inputs | Tool modifies inputs in-place |

Shadow directories are created on the filesystem where `.snakemake/` lives. On HPC clusters, this is typically shared NFS, so shadow works across Slurm nodes without extra configuration. If `.snakemake/` is on node-local storage, use `--shadow-prefix` to point to shared storage.

---

## localrules

Mark lightweight rules that should run on the head node, not as Slurm jobs:

```python
localrules: all, download_reference, plot_summary, create_config

rule all:
    input: expand("results/{sample}.bam", sample=SAMPLES)

rule download_reference:
    output: "ref/genome.fa"
    shell: "wget -O {output} {GENOME_URL}"

rule plot_summary:
    input: "results/stats.csv"
    output: "results/summary.png"
    script: "scripts/plot.py"
```

**Good candidates for localrules:** target rules (`all`), downloads, plotting, file moves, metadata extraction, config generation.

---

## Grouping Jobs

Reduce Slurm job submission overhead by grouping small rules into single jobs:

```python
# In Snakefile
rule extract_info:
    group: "preprocessing"
    ...

rule filter_reads:
    group: "preprocessing"
    ...
```

Or via CLI:
```bash
snakemake --groups extract_info=preprocess filter_reads=preprocess \
          --group-components preprocess=10
```

`--group-components preprocess=10` packs up to 10 instances of the group into one Slurm job.

> **Warning: Numeric partition names break group jobs.** Snakemake's group resource calculator sums `slurm_partition` values across grouped rules when the partition name is numeric (e.g., 5 rules with partition `20` → sbatch gets `-p 100`). This is a known upstream bug. **Workaround:** Use automatic partition selection via `slurm-partition-config: partitions.yaml` instead of setting `slurm_partition` directly. See [Automatic Partition Selection](#automatic-partition-selection).

---

## NFS / Shared Filesystem

Shared filesystems have propagation delays. Snakemake may think output files are missing when they exist but haven't propagated yet.

```yaml
# In profile
latency-wait: 60          # seconds to wait before rechecking
rerun-triggers: mtime      # rerun based on modification time, not existence
```

Increase `latency-wait` for large/slow NFS mounts. 60 seconds is a reasonable default; some setups need more.

---

## Debugging Failed Jobs

### Check job status

```bash
# Your active jobs
squeue --me

# Detailed job info after completion
sacct -j JOBID --format=JobID,JobName,State,ExitCode,MaxRSS,Elapsed,Timelimit

# Resource efficiency report
seff JOBID
```

### Common failure patterns

| Symptom | Likely cause | Fix |
|---|---|---|
| `OUT_OF_MEMORY` | Requested too little memory | Increase `mem_mb`, or use `attempt`-scaled resources with retries |
| `TIMEOUT` | Walltime exceeded | Increase `runtime` |
| `MissingOutputException` | NFS delay | Increase `latency-wait` |
| `CANCELLED` | Preemption or manual cancel | Use `--slurm-requeue` for auto-requeue |
| Job stuck in `PENDING` | Partition full or resource request too large | Check `sinfo`, reduce resources or try different partition |
| `sbatch: error: Batch job submission failed` | Account/partition misconfigured | Verify `slurm_account` and `slurm_partition` in profile |
| `sbatch: error: invalid partition` (group jobs only) | Numeric partition name summed | Use `slurm-partition-config` instead of `slurm_partition` |

### Find Slurm log files

By default, the executor plugin writes logs to `.snakemake/slurm_logs/`. To customize:

```yaml
# In profile
default-resources:
    slurm_extra: "'--output=logs/slurm/%j-%x.out --error=logs/slurm/%j-%x.err'"
```

### Rerun failed jobs

```bash
# Rerun only incomplete/failed
snakemake --profile slurm/ --rerun-incomplete

# Rerun specific targets
snakemake --profile slurm/ --forcerun rule_name
```

---

## Useful Plugin Flags

| Flag | Purpose |
|---|---|
| `--slurm-requeue` | Auto-requeue preempted jobs |
| `--slurm-exclude-failed-nodes` | Blacklist nodes that caused failures |
| `--slurm-keep-successful-logs` | Don't delete logs from successful jobs |
| `--slurm-pass-command-as-script` | Work around `--wrap` length limits |
| `--slurm-jobname-prefix` | Readable prefix in `squeue` output |
| `--slurm-no-account` | Disable account auto-detection |
| `--slurm-init-seconds-before-status-checks` | Delay before first `sacct` poll (default 40s) |

---

## Software Environments

Snakemake rules need access to the right tools. There are several approaches — use whatever fits your project:

### Snakemake-managed conda (`--use-conda`)

Snakemake creates and caches per-rule conda environments automatically:

```python
rule align:
    input: "reads.fastq"
    output: "aligned.bam"
    conda: "envs/align.yaml"
    shell: "bwa mem ..."
```

### Pre-built environments (conda, uv, venv)

Activate an existing environment before calling snakemake, or activate within rules:

```python
rule analyze:
    input: "data.h5"
    output: "results.csv"
    shell:
        """
        source activate myenv   # conda
        # or: source .venv/bin/activate  # venv/uv
        python scripts/analyze.py {input} {output}
        """
```

### Environment modules (`--use-envmodules`)

Common on HPC clusters with Lmod or Environment Modules:

```python
rule align:
    input: "reads.fastq"
    output: "aligned.bam"
    envmodules: "bwa/0.7.18", "samtools/1.20"
    shell: "bwa mem ..."
```

### Containers (`--use-singularity`)

```python
rule align:
    input: "reads.fastq"
    output: "aligned.bam"
    container: "docker://biocontainers/bwa:0.7.18"
    shell: "bwa mem ..."
```

Choose whichever approach your project already uses. The Slurm executor works with all of them.

---

## Project Structure

```
project/
├── workflow/
│   ├── Snakefile
│   ├── rules/
│   │   ├── preprocessing.smk
│   │   ├── alignment.smk
│   │   └── analysis.smk
│   ├── envs/
│   │   ├── align.yaml
│   │   └── analysis.yaml
│   └── scripts/
│       ├── process.py
│       └── plot.R
├── config/
│   └── config.yaml
├── slurm/
│   └── config.yaml          # Slurm profile
└── results/
```

Include rules from separate files:

```python
# Snakefile
configfile: "config/config.yaml"

include: "rules/preprocessing.smk"
include: "rules/alignment.smk"
include: "rules/analysis.smk"
```

---

## Quick Reference

```bash
# Submit workflow
snakemake --workflow-profile slurm/

# Dry run (see what would be submitted)
snakemake --workflow-profile slurm/ -n

# DAG visualization
snakemake --dag | dot -Tpng > dag.png

# Monitor jobs
squeue --me
watch -n 30 'squeue --me | wc -l'

# Cancel all your Snakemake jobs
scancel --me

# Check resource efficiency of completed jobs
sacct --me -S today --format=JobID,JobName,State,MaxRSS,Elapsed,Timelimit,ExitCode
```

---

## Critical Rules

### Never

- **Hardcode partitions/accounts in Snakefiles** — use profiles for portability
- **Run Snakemake inside `salloc`/`srun`** — it submits its own jobs; run on head node
- **Set both `mem_mb` and `mem_mb_per_cpu`** — mutually exclusive Slurm flags
- **Skip `latency-wait` on shared filesystems** — NFS delays cause false "missing output" errors
- **Use legacy `--cluster` or `--cluster-config`** — deprecated in Snakemake 8
- **Set numeric `slurm_partition` with group jobs** — values get summed; use `slurm-partition-config` instead
- **Rely on `runtime` in profile `default-resources`** — it may not propagate; set `runtime` in rule `resources:` blocks

### Always

- **Use `localrules`** for lightweight tasks (targets, downloads, plotting)
- **Use `--keep-going`** so one failed job doesn't block independent branches
- **Use `attempt`-scaled resources** with `--retries` for memory-hungry rules
- **Run in tmux/screen** — the scheduler must stay alive for the duration and for `temp()` cleanup
- **Log your runs** — pipe to `tee` with timestamps for debugging later
- **Use `temp()`** on intermediate files to avoid filling shared storage
- **Use `protected()`** on expensive-to-reproduce outputs to prevent accidental overwrite
