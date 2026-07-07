# 4. HPC Integration

## Why Your Laptop Is Not Enough

With 6 samples, your pipeline finishes quickly on a workstation. A real experiment with 100 samples tells a different story. The STAR alignment step alone requires 8 threads per sample. A 16-core laptop can run at most 2 STAR jobs in parallel, meaning 50 sequential rounds of alignment, easily many hours of wall time.

Submitting to a SLURM cluster lets Snakemake dispatch up to 100 jobs simultaneously, each on its own compute node, while it sits on the login node managing the queue. The Snakefile does not change at all.

## The SLURM Executor Plugin

Snakemake 8 uses an executor plugin model. Install the SLURM plugin inside your `snakemake_workshop` conda environment:

```bash
conda activate snakemake_workshop
pip install snakemake-executor-plugin-slurm
```

## Running on SLURM

The only change from a local run is the flags you pass on the command line:

=== "Local"
    ```bash
    snakemake --cores 16
    ```

=== "SLURM cluster"
    ```bash
    snakemake \
        --executor slurm \
        --jobs 100 \
        --default-resources \
            slurm_account=your_account \
            slurm_partition=cpu \
            mem_mb=8000 \
            runtime=60
    ```

Replace `your_account` with your UNR HPC allocation name. With `--jobs 100`, Snakemake keeps up to 100 SLURM jobs in the queue simultaneously.

!!! note "Finding your UNR SLURM account"
    Run the following on the cluster login node to see your active allocation(s):
    ```bash
    sacctmgr show user $USER withassoc format=account
    ```
    Contact the [UNR Research Computing](https://www.unr.edu/research-computing) team if you are unsure which account to use.

## How Resource Directives Map to SLURM

The `threads:` and `resources:` values you set in each rule are automatically translated into SLURM `#SBATCH` directives,  no changes to the Snakefile required:

| Snakemake directive | SLURM `#SBATCH` equivalent |
|---|---|
| `threads: 8` | `--cpus-per-task=8` |
| `resources: mem_mb=16000` | `--mem=16000M` |
| `resources: runtime=120` | `--time=120` (minutes) |
| `--default-resources slurm_partition=cpu` | `--partition=cpu` |
| `--default-resources slurm_account=your_account` | `--account=your_account` |

Rules without an explicit `threads:` use 1 CPU. Rules without explicit `mem_mb` or `runtime` inherit the `--default-resources` values (8 GB RAM, 60 minutes). Your `rule star_align` will automatically request 8 CPUs, 16 GB, and 60 minutes; all other rules will request 1 CPU, 8 GB, and 60 minutes. If a rule needs more time, override it with `resources: runtime=120`.

## Using a Snakemake Profile

Typing all the SLURM flags on the command line every time is tedious and error-prone. A **profile** is a YAML configuration file stored in `~/.config/snakemake/` that sets default options for a named execution environment.

Create the profile directory and configuration file:

```bash
mkdir -p ~/.config/snakemake/slurm
```

Save the following as `~/.config/snakemake/slurm/config.yaml`, substituting your real account name:

```yaml
executor: slurm
jobs: 100
default-resources:
  slurm_account: your_account
  slurm_partition: cpu
  mem_mb: 8000
  runtime: 60
latency-wait: 60
rerun-incomplete: true
```

Now the entire cluster submission is a single command:

```bash
snakemake --profile slurm
```

`latency-wait: 60` gives shared filesystems up to 60 seconds to make newly created files visible, important on NFS-mounted cluster storage where there can be a brief delay between a file being written and becoming visible to other nodes.

## Monitoring Your Jobs

While Snakemake is running, open a second terminal on the login node to watch the queue:

```bash
squeue -u $USER
```

Snakemake writes a detailed progress log to `.snakemake/log/`. If a job fails, its SLURM stdout/stderr is stored alongside the Snakemake log, identified by job ID.

## Handling Failures and Re-running

If a run is interrupted, a node crashes, walltime is exceeded, or you cancel it, Snakemake marks any partially-written output files as incomplete. On the next run, Snakemake will refuse to overwrite them unless you explicitly request it:

```bash
snakemake --profile slurm --rerun-incomplete
```

This deletes the incomplete outputs and re-submits only the affected jobs, leaving all successfully completed outputs untouched.

!!! tip "Testing on the cluster before full submission"
    Use a dry-run (`-n`) with the SLURM profile to verify Snakemake can see all inputs and has a valid execution plan before spending allocation time:
    ```bash
    snakemake --profile slurm -n
    ```

!!! note "Where we are"
    Your Snakefile is now cluster-ready with zero modifications. The resource declarations from the previous section do double duty: controlling local parallelism on a laptop **and** determining SLURM job specifications on the cluster.

    The final step pins the software environment of every rule so this pipeline produces byte-for-byte reproducible results on any machine, now and in the future. Continue to [Reproducibility](reproducibility.md).
