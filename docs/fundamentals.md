# 1. Snakemake Fundamentals

## The Problem with Shell Scripts

You have your FASTQ files ready. The natural first move is a shell script:

```bash
fastqc data/wt_rep1_R1.fastq.gz  --outdir results/fastqc/
fastqc data/wt_rep1_R2.fastq.gz  --outdir results/fastqc/
fastqc data/wt_rep2_R1.fastq.gz  --outdir results/fastqc/
fastqc data/wt_rep2_R2.fastq.gz  --outdir results/fastqc/
fastqc data/wt_rep3_R1.fastq.gz  --outdir results/fastqc/
fastqc data/wt_rep3_R2.fastq.gz  --outdir results/fastqc/
fastqc data/mut_rep1_R1.fastq.gz --outdir results/fastqc/
fastqc data/mut_rep1_R2.fastq.gz --outdir results/fastqc/
fastqc data/mut_rep2_R1.fastq.gz --outdir results/fastqc/
fastqc data/mut_rep2_R2.fastq.gz --outdir results/fastqc/
fastqc data/mut_rep3_R1.fastq.gz --outdir results/fastqc/
fastqc data/mut_rep3_R2.fastq.gz --outdir results/fastqc/
```

This works for 6 samples. With 50 samples it becomes 100 lines to maintain. It re-runs everything every time, even steps that already finished. It can't parallelise jobs automatically. And it has no concept of dependencies ... if trimming fails, downstream alignment still tries to run.

**Snakemake solves all of this.** You describe *what* to produce and the relationship between steps. Snakemake works out the order, skips steps whose outputs are already up to date, and runs independent steps in parallel.

## The Snakefile

A `Snakefile` is a plain text file that Snakemake reads. It lives in your project root. Create one now:

```bash
cd snakemake_tutorial   # your project directory from the Data Setup page
touch Snakefile
```

## Anatomy of a Rule

Everything in Snakemake is built from **rules**. A rule describes one step in your pipeline:

```python
rule rule_name:        # (1) A name you choose - no spaces
    input:             # (2) The file(s) this step needs
        "path/to/input_file"
    output:            # (3) The file(s) this step will create
        "path/to/output_file"
    shell:             # (4) The command to run
        "some_tool {input} > {output}"
```

`{input}` and `{output}` inside the `shell:` string are **placeholders**. Snakemake substitutes the actual paths at runtime. You never repeat the filenames.

## Write Your First Rule

Open your `Snakefile` and add the following:

```python
rule fastqc:
    input:
        "data/wt_rep1_R1.fastq.gz"
    output:
        html="results/fastqc/wt_rep1_R1_fastqc.html",
        zip= "results/fastqc/wt_rep1_R1_fastqc.zip"
    shell:
        "fastqc {input} --outdir results/fastqc/"
```

This rule runs FastQC on a single file and produces two outputs, an HTML report and a ZIP archive. Both output files are **named** (`html` and `zip`) so they can be referred to individually later if needed.

## Running Snakemake

Create the output directory, then ask Snakemake to produce the HTML report:

```bash
mkdir -p results/fastqc
snakemake --cores 1 results/fastqc/wt_rep1_R1_fastqc.html
```

Snakemake prints the rule it is running and the exact shell command, then exits with a success message. Confirm the output exists:

```bash
ls results/fastqc/
```

### The Key Mental Model: Working Backwards

Snakemake never reads your Snakefile top to bottom like a shell script. Instead it works **backwards from the target you requested**:

1. *"Which rule produces `results/fastqc/wt_rep1_R1_fastqc.html`?"* → `rule fastqc`
2. *"Does that rule's input (`data/wt_rep1_R1.fastq.gz`) exist?"* → Yes
3. *"Is the output already newer than the input?"* → No → **run the rule**

Run the same command a second time:

```bash
snakemake --cores 1 results/fastqc/wt_rep1_R1_fastqc.html
```

Snakemake responds: **"Nothing to be done."** The output already exists and is newer than its input, so the rule is skipped. This is the foundation of efficient pipeline re-running.

!!! tip "Always do a dry-run first"
    Add `-n` to preview what Snakemake *would* do without running anything:
    ```bash
    snakemake -n results/fastqc/wt_rep1_R1_fastqc.html
    ```
    This prints each planned job with its rule name and command. Use it to catch errors before committing to a long run.

## Visualising the DAG

Snakemake can draw the dependency graph (DAG) of your workflow. Install Graphviz if needed:

```bash
mamba install -c conda-forge graphviz
```

Then generate and open the graph:

```bash
snakemake --dag dot results/fastqc/wt_rep1_R1_fastqc.html | dot -Tsvg > dag.svg
```

Right now it shows a single node for `rule fastqc`. As you add rules in later sections, this graph will grow to represent the full pipeline and Snakemake uses this graph to determine what to run in parallel.

## The `rule all` Convention

Instead of typing a target filename on the command line every time, Snakemake uses a convention: the **first rule** in the Snakefile is the default target. By convention this rule is called `rule all`, and its `input:` lists all the final outputs you want.

Update your Snakefile so `rule all` sits at the very top:

```python
rule all:
    input:
        "results/fastqc/wt_rep1_R1_fastqc.html",
        "results/fastqc/wt_rep1_R1_fastqc.zip"

rule fastqc:
    input:
        "data/wt_rep1_R1.fastq.gz"
    output:
        html="results/fastqc/wt_rep1_R1_fastqc.html",
        zip= "results/fastqc/wt_rep1_R1_fastqc.zip"
    shell:
        "fastqc {input} --outdir results/fastqc/"
```

Now running the entire pipeline is simply:

```bash
snakemake --cores 1
```

Snakemake sees `rule all` first, looks at its inputs, finds the rules that produce them, and runs the necessary steps in order.

!!! note "Where we are"
    You have a working Snakefile that runs FastQC on one file. The limitation is obvious: to process all 6 FASTQ files across 3 samples you would need 6 copies of this rule, identical except for the filename. That is exactly the problem **wildcards** solve.

    Continue to [Wildcards & Generalization](wildcards.md).
