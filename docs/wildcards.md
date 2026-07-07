# 2. Wildcards & Generalization

## The Copy-Paste Problem

At the end of the last section, your `rule fastqc` processes exactly one file: `wt_rep1_R1.fastq.gz`. To cover all 12 FASTQ files across 6 samples and 2 read directions, you would need to copy that rule 12 times, changing only the filename each time. With 50 samples, that is 100 nearly-identical rules ... a maintenance nightmare.

Snakemake solves this with **wildcards**: named placeholders inside file paths that Snakemake fills in automatically based on which output file was requested.

## Introducing Wildcards

A wildcard is written as `{name}` anywhere inside an `input:` or `output:` path. Snakemake infers the wildcard's value by pattern-matching against the filename of the output you requested.

Replace the hardcoded `rule fastqc` in your Snakefile with this generalised version:

```python
rule fastqc:
    input:
        "data/{sample}_{read}.fastq.gz"
    output:
        html="results/fastqc/{sample}_{read}_fastqc.html",
        zip= "results/fastqc/{sample}_{read}_fastqc.zip"
    shell:
        "fastqc {input} --outdir results/fastqc/"
```

Now, if you ask Snakemake to produce `results/fastqc/wt_rep2_R2_fastqc.html`, it automatically:

1. Matches `{sample}` = `wt_rep2` and `{read}` = `R2`
2. Looks for input `data/wt_rep2_R2.fastq.gz`
3. Runs FastQC on that file

**One rule handles every combination of sample and read direction.**

## The `expand()` Function

With wildcards in the rules, you need a way to tell Snakemake *which specific combinations* to generate. That is the job of `expand()`.

Define your samples and read directions at the top of the Snakefile, then use `expand()` in `rule all`:

```python
SAMPLES = ["wt_rep1", "wt_rep2", "wt_rep3", "mut_rep1", "mut_rep2", "mut_rep3"]
READS   = ["R1", "R2"]

rule all:
    input:
        expand("results/fastqc/{sample}_{read}_fastqc.html",
               sample=SAMPLES, read=READS)
```

`expand()` generates every combination of sample and read direction. With six samples and two reads, that is twelve target files produced from two short lists.

!!! warning "Wildcards vs. `expand()` - know where each belongs"
    - **Wildcards** (`{sample}`) go inside **rules** - in `input:` and `output:` blocks. They tell Snakemake *how to generalise* a rule.
    - **`expand()`** goes in **`rule all`** (and other places where you list specific target files). It tells Snakemake *which files to produce*.

    You cannot put a bare wildcard directly in `rule all`'s input. Snakemake would have no way to know which values to substitute.

## Adding `rule fastp` (Trim)

Now add the trimming step. fastp processes both reads of a sample in a single command and produces JSON and HTML QC reports that MultiQC will aggregate later.

Add this rule below `rule fastqc`:

```python
rule fastp:
    input:
        r1="data/{sample}_R1.fastq.gz",
        r2="data/{sample}_R2.fastq.gz"
    output:
        r1=  "results/trimmed/{sample}_R1.fastq.gz",
        r2=  "results/trimmed/{sample}_R2.fastq.gz",
        json="results/trimmed/{sample}_fastp.json",
        html="results/trimmed/{sample}_fastp.html"
    shell:
        """
        fastp \
            --in1 {input.r1}  --in2 {input.r2} \
            --out1 {output.r1} --out2 {output.r2} \
            --json {output.json} --html {output.html}
        """
```

Note the **named inputs and outputs**: `input.r1`, `output.json`, etc. When a rule has multiple inputs or outputs, naming them makes it straightforward to reference each one correctly inside the shell command.

Unlike `rule fastqc`, which runs on each read file independently, `rule fastp` uses only the `{sample}` wildcard and processes both read directions together. Snakemake handles the paired-end logic because you explicitly named `input.r1` and `input.r2`.

## Your Complete Snakefile So Far

Update your Snakefile to match the following. Note that `rule all` now requests both FastQC and trimmed outputs:

```python
SAMPLES = ["wt_rep1", "wt_rep2", "wt_rep3", "mut_rep1", "mut_rep2", "mut_rep3"]
READS   = ["R1", "R2"]

rule all:
    input:
        expand("results/fastqc/{sample}_{read}_fastqc.html",
               sample=SAMPLES, read=READS),
        expand("results/trimmed/{sample}_{read}.fastq.gz",
               sample=SAMPLES, read=READS)

rule fastqc:
    input:
        "data/{sample}_{read}.fastq.gz"
    output:
        html="results/fastqc/{sample}_{read}_fastqc.html",
        zip= "results/fastqc/{sample}_{read}_fastqc.zip"
    shell:
        "fastqc {input} --outdir results/fastqc/"

rule fastp:
    input:
        r1="data/{sample}_R1.fastq.gz",
        r2="data/{sample}_R2.fastq.gz"
    output:
        r1=  "results/trimmed/{sample}_R1.fastq.gz",
        r2=  "results/trimmed/{sample}_R2.fastq.gz",
        json="results/trimmed/{sample}_fastp.json",
        html="results/trimmed/{sample}_fastp.html"
    shell:
        """
        fastp \
            --in1 {input.r1}  --in2 {input.r2} \
            --out1 {output.r1} --out2 {output.r2} \
            --json {output.json} --html {output.html}
        """
```

## Running the Two-Step Pipeline

Do a dry-run to review the execution plan before running anything:

```bash
snakemake -n
```

You will see 19 planned jobs: 12 FastQC jobs (one per read file), 6 fastp jobs (one per sample), and `rule all`. Then run for real:

```bash
mkdir -p results/trimmed
snakemake --cores 4
```

Snakemake parallelises the independent FastQC and fastp jobs up to the `--cores` limit. Verify the outputs:

```bash
ls results/fastqc/    # 12 HTML + 12 ZIP files
ls results/trimmed/   # 12 trimmed FASTQ + 6 JSON + 6 HTML files
```

!!! note "Where we are"
    Your Snakefile now handles all 6 samples and both read directions with just two rules. Adding a fourth sample is as simple as appending its name to the `SAMPLES` list, no other changes needed.

    The next step introduces STAR alignment and featureCounts quantification, and teaches Snakemake how to schedule those resource-intensive jobs without overwhelming your machine. Continue to [Resource Management](resources.md).
