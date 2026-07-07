# 5. Reproducibility

## Why Software Versions Matter

A different version of STAR can produce different alignment rates. A different version of featureCounts can assign reads differently depending on its default overlap-resolution behaviour. These are not hypothetical concerns, they have caused irreproducible published results. Without pinning the exact software used in your pipeline, reproducing your analysis six months, or six years, later is guesswork.

Snakemake's `conda:` directive solves this cleanly. Each rule declares its own isolated Conda environment in a version-locked YAML file. Snakemake creates these environments automatically the first time the pipeline runs and reuses them on every subsequent run, on your laptop, the cluster, or a collaborator's machine.

## Creating the Environment Files

Create an `envs/` directory in your project:

```bash
mkdir -p envs
```

Save one YAML file per rule. Each file pins the tool to a specific version and lists the Conda channels to search:

=== "envs/fastqc.yaml"
    ```yaml
    channels:
      - bioconda
      - conda-forge
    dependencies:
      - fastqc=0.12.1
    ```

=== "envs/fastp.yaml"
    ```yaml
    channels:
      - bioconda
      - conda-forge
    dependencies:
      - fastp=0.23.4
    ```

=== "envs/star.yaml"
    ```yaml
    channels:
      - bioconda
      - conda-forge
    dependencies:
      - star=2.7.11a
    ```

=== "envs/subread.yaml"
    ```yaml
    channels:
      - bioconda
      - conda-forge
    dependencies:
      - subread=2.0.6
    ```

=== "envs/multiqc.yaml"
    ```yaml
    channels:
      - bioconda
      - conda-forge
    dependencies:
      - multiqc=1.21
    ```

## Adding `conda:` Directives to Each Rule

Add a `conda:` line to every rule that runs an external tool. The path is relative to the Snakefile location:

```python
rule fastqc:
    input:
        "data/{sample}_{read}.fastq.gz"
    output:
        html="results/fastqc/{sample}_{read}_fastqc.html",
        zip= "results/fastqc/{sample}_{read}_fastqc.zip"
    conda:
        "envs/fastqc.yaml"
    shell:
        "fastqc {input} --outdir results/fastqc/"
```

Apply the same pattern to all remaining rules:

| Rule | `conda:` value |
|---|---|
| `rule fastqc` | `"envs/fastqc.yaml"` |
| `rule fastp` | `"envs/fastp.yaml"` |
| `rule star_align` | `"envs/star.yaml"` |
| `rule featurecounts` | `"envs/subread.yaml"` |
| `rule multiqc` | `"envs/multiqc.yaml"` |

`rule all` does not run any tools, so it does not need a `conda:` directive.

## Running with `--use-conda`

```bash
snakemake --cores 8 --use-conda
```

On the first run, Snakemake creates each Conda environment and caches it in `.snakemake/conda/`. This takes a few minutes per environment the first time. On every subsequent run, the cached environments are reused instantly.

!!! tip "Combining with SLURM"
    `--use-conda` works seamlessly with the SLURM profile from the previous section. Each cluster job activates its own isolated environment before running:
    ```bash
    snakemake --profile slurm --use-conda
    ```

## Sharing Your Workflow

Push the following files to GitHub and anyone, a collaborator, a reviewer, or your future self, can reproduce your entire analysis with two commands:

```
snakemake_tutorial/
├── Snakefile
├── envs/
│   ├── fastqc.yaml
│   ├── fastp.yaml
│   ├── star.yaml
│   ├── subread.yaml
│   └── multiqc.yaml
└── README.md
```

```bash
git clone https://github.com/NevadaINBRE/snakemake_tutorial.git
cd snakemake_tutorial
bash setup.sh            # download data + build STAR index
snakemake --cores 8 --use-conda
```

No installation guide. No "it depends on your system." No missing dependencies.

!!! note "Container alternative: Apptainer"
    If you need even stronger isolation, for example, pinning system C libraries alongside Conda packages, Snakemake also supports Apptainer (formerly Singularity) containers via `--use-apptainer` and a `container:` directive per rule. Apptainer is available on most HPC clusters where Docker is not permitted, and is a common choice for production workflows submitted to journals as supplementary materials.

## Complete Final Snakefile

??? note "Click to expand the complete Snakefile with conda directives"

    ```python
    SAMPLES    = ["wt_rep1", "wt_rep2", "wt_rep3", "mut_rep1", "mut_rep2", "mut_rep3"]
    READS      = ["R1", "R2"]
    GENOME_DIR = "genome/star_index"
    GTF        = "genome/Saccharomyces_cerevisiae.R64-1-1.gtf"

    rule all:
        input:
            "results/multiqc/multiqc_report.html"

    rule fastqc:
        input:
            "data/{sample}_{read}.fastq.gz"
        output:
            html="results/fastqc/{sample}_{read}_fastqc.html",
            zip= "results/fastqc/{sample}_{read}_fastqc.zip"
        conda:
            "envs/fastqc.yaml"
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
            expand("results/aligned/{sample}.Log.final.out", sample=SAMPLES),
            "results/counts/all_samples.txt"
        output:
            "results/multiqc/multiqc_report.html"
        conda:
            "envs/multiqc.yaml"
        shell:
            "multiqc results/ --outdir results/multiqc/ --force"
    ```
