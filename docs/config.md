# 6. Config Files

## The Problem with Hardcoded Values

Your Snakefile currently opens with four hardcoded variables:

```python
SAMPLES    = ["wt_rep1", "wt_rep2", "wt_rep3", "mut_rep1", "mut_rep2", "mut_rep3"]
READS      = ["R1", "R2"]
GENOME_DIR = "genome/star_index"
GTF        = "genome/Saccharomyces_cerevisiae.R64-1-1.gtf"
```

To reuse this pipeline on a different dataset, a collaborator must open the Snakefile and edit it directly. That breaks a core principle of reproducible workflows: the **logic** of the pipeline (the rules) should be completely separate from the **configuration** (the data paths and parameters).

Snakemake solves this with the `configfile:` directive and the `glob_wildcards()` function.

## Creating a Config File

A Snakemake config file is a plain YAML file. Create `config.yaml` in your project root:

```yaml
reads_dir: "data"           # directory containing your FASTQ files
reads:                      # read-pair suffixes
  - "R1"
  - "R2"
extension: "fastq.gz"       # file extension (without leading dot)
fasta: "genome/Saccharomyces_cerevisiae.R64-1-1.dna.fa"  # reference FASTA
genome_dir: "genome/star_index"                           # STAR index directory
gtf: "genome/Saccharomyces_cerevisiae.R64-1-1.gtf"
index_extra: "--genomeSAindexNbases 10"  # extra STAR index flags; adjust for genome size
```

To run the same pipeline on a new dataset, a user edits only this file. The Snakefile itself never changes.

## Using `configfile:` in the Snakefile

Add a single `configfile:` directive at the very top of your Snakefile. Snakemake loads the YAML and makes its contents available as a dictionary called `config`:

```python
configfile: "config.yaml"
```

You can then reference any key with `config["key"]`.

## Auto-discovering Samples with `glob_wildcards()`

Instead of a hardcoded `SAMPLES` list, use Snakemake's built-in `glob_wildcards()` to scan the reads directory and extract sample names automatically. It applies a pattern to all matching files and returns the values captured by each wildcard.

Replace the old variable block at the top of your Snakefile with:

```python hl_lines="1 9"
configfile: "config.yaml" # (1)!

READS_DIR  = config["reads_dir"]
READS      = config["reads"]
EXT        = config["extension"]
FASTA       = config["fasta"]
GENOME_DIR  = config["genome_dir"]
GTF         = config["gtf"]
INDEX_EXTRA = config["index_extra"]

SAMPLES = glob_wildcards(READS_DIR + "/{sample}_" + READS[0] + "." + EXT).sample # (2)!
```

1. `configfile:` is a Snakemake keyword, not a Python statement. Place it before any Python code.
2. `glob_wildcards()` scans the filesystem for files matching the pattern and returns a namedtuple. The `.sample` attribute is the list of all values captured by `{sample}`. Using `READS[0]` (i.e., `"R1"`) ensures each sample is counted once, from its forward read.

For our dataset, this produces the same list as before:

```
["wt_rep1", "wt_rep2", "wt_rep3", "mut_rep1", "mut_rep2", "mut_rep3"]
```

But it will work equally well if you drop new samples into the `data/` directory and rerun, with no Snakefile edits needed.

!!! tip "Sorting the sample list"
    `glob_wildcards()` returns samples in filesystem order, which may vary across machines. For reproducible job ordering, sort the result:
    ```python
    SAMPLES = sorted(glob_wildcards(READS_DIR + "/{sample}_" + READS[0] + "." + EXT).sample)
    ```

## Adding a STAR Index Rule

Previously, students built the STAR index manually in `setup.sh`. Now that the FASTA path and index parameters live in `config.yaml`, you can let Snakemake build the index automatically as part of the pipeline. It will only run if the index directory does not already exist.

Snakemake uses the `directory()` wrapper to declare that a rule produces a folder rather than a single file:

```python
rule star_index:
    input:
        fasta=FASTA,
        gtf=GTF
    output:
        directory(GENOME_DIR) # (1)!
    threads: 4
    resources:
        mem_mb=8000
    conda:
        "envs/star.yaml"
    shell:
        """
        STAR \
            --runMode genomeGenerate \
            --runThreadN {threads} \
            --genomeDir {output} \
            --genomeFastaFiles {input.fasta} \
            --sjdbGTFfile {input.gtf} \
            {INDEX_EXTRA}
        """
```

1. `directory()` tells Snakemake the output is a folder. Without it, Snakemake would look for a file at that path and never consider the rule satisfied.

The `star_align` rule already lists `index=GENOME_DIR` as an input. Because `star_index` declares `directory(GENOME_DIR)` as its output, Snakemake automatically connects the two rules: alignment will not start until the index is built.

!!! tip "Why `index_extra` belongs in config"
    STAR's `--genomeSAindexNbases` flag must match the genome size. The default (`14`) is tuned for human (~3 GB). The yeast genome (~12 MB) needs `10`. Storing extra flags as a single string in `config.yaml` means students working on a different organism change one value instead of hunting through the Snakefile. It also makes it easy to add other indexing flags later without touching any rules.

## Running with a Config Override

You can override any config value on the command line with `--config`, without editing `config.yaml`:

```bash
snakemake --cores 8 --use-conda --config reads_dir=other_data/
```

This is useful for quick testing on a subset directory or switching datasets without touching any files.

## Complete Updated Snakefile

??? note "Click to expand the complete Snakefile with configfile and glob_wildcards"

    ```python
    configfile: "config.yaml"

    READS_DIR  = config["reads_dir"]
    READS      = config["reads"]
    EXT        = config["extension"]
    FASTA       = config["fasta"]
    GENOME_DIR  = config["genome_dir"]
    GTF         = config["gtf"]
    INDEX_EXTRA = config["index_extra"]

    SAMPLES = sorted(glob_wildcards(READS_DIR + "/{sample}_" + READS[0] + "." + EXT).sample)

    rule all:
        input:
            "results/multiqc/multiqc_report.html"

    rule star_index:
        input:
            fasta=FASTA,
            gtf=GTF
        output:
            directory(GENOME_DIR)
        threads: 4
        resources:
            mem_mb=8000
        conda:
            "envs/star.yaml"
        shell:
            """
            STAR \\
                --runMode genomeGenerate \\
                --runThreadN {threads} \\
                --genomeDir {output} \\
                --genomeFastaFiles {input.fasta} \\
                --sjdbGTFfile {input.gtf} \\
                {INDEX_EXTRA}
            """

    rule fastqc:
        input:
            READS_DIR + "/{sample}_{read}." + EXT
        output:
            html="results/fastqc/{sample}_{read}_fastqc.html",
            zip= "results/fastqc/{sample}_{read}_fastqc.zip"
        conda:
            "envs/fastqc.yaml"
        shell:
            "fastqc {input} --outdir results/fastqc/"

    rule fastp:
        input:
            r1=READS_DIR + "/{sample}_" + READS[0] + "." + EXT,
            r2=READS_DIR + "/{sample}_" + READS[1] + "." + EXT
        output:
            r1=  "results/trimmed/{sample}_R1.fastq.gz",
            r2=  "results/trimmed/{sample}_R2.fastq.gz",
            json="results/trimmed/{sample}_fastp.json",
            html="results/trimmed/{sample}_fastp.html"
        conda:
            "envs/fastp.yaml"
        shell:
            """
            fastp \
                --in1 {input.r1}  --in2 {input.r2} \
                --out1 {output.r1} --out2 {output.r2} \
                --json {output.json} --html {output.html}
            """

    rule star_align:
        input:
            r1=   "results/trimmed/{sample}_R1.fastq.gz",
            r2=   "results/trimmed/{sample}_R2.fastq.gz",
            index=GENOME_DIR
        output:
            bam="results/aligned/{sample}.Aligned.sortedByCoord.out.bam",
            log="results/aligned/{sample}.Log.final.out"
        threads: 8
        resources:
            mem_mb=16000
        conda:
            "envs/star.yaml"
        shell:
            """
            mkdir -p results/aligned
            STAR \
                --runThreadN {threads} \
                --genomeDir {input.index} \
                --readFilesIn {input.r1} {input.r2} \
                --readFilesCommand zcat \
                --outSAMtype BAM SortedByCoordinate \
                --outFileNamePrefix results/aligned/{wildcards.sample}. \
                --outSAMattributes NH HI AS NM MD
            """

    rule featurecounts:
        input:
            bams=expand("results/aligned/{sample}.Aligned.sortedByCoord.out.bam",
                        sample=SAMPLES),
            gtf=GTF
        output:
            counts="results/counts/all_samples.txt"
        threads: 4
        conda:
            "envs/subread.yaml"
        shell:
            """
            mkdir -p results/counts
            featureCounts \
                -T {threads} \
                -p \
                -a {input.gtf} \
                -o {output.counts} \
                {input.bams}
            """

    rule multiqc:
        input:
            expand("results/fastqc/{sample}_{read}_fastqc.zip",
                   sample=SAMPLES, read=READS),
            expand("results/trimmed/{sample}_fastp.json", sample=SAMPLES),
            expand("results/aligned/{sample}.Log.final.out", sample=SAMPLES)
        output:
            "results/multiqc/multiqc_report.html"
        conda:
            "envs/multiqc.yaml"
        shell:
            "multiqc results/ --outdir results/multiqc --force"
    ```

!!! note "Where we are"
    Your Snakefile is now fully portable. Swapping datasets means editing one YAML file, not the pipeline itself. The final section shows how to replace per-rule Conda environments with a single Docker container for maximum reproducibility. Continue to [Docker Containers](docker.md).
