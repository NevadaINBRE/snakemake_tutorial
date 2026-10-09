# 7. Docker Containers

## From Conda to Containers

In the previous section we pinned software by giving each rule its own `conda:` environment file. That approach works extremely well on a single machine, but it has a subtle limitation: Conda only controls the *user-space* software stack. System C libraries (glibc, libstdc++, OpenSSL) are inherited from the host operating system. In most cases this is fine; occasionally, a tool will behave differently, or refuse to run, because the host OS is years older or newer than the one the package was built against.

A Docker container packages the tool *and the entire operating system layer* it was built on. The result is a single portable artifact that is byte-for-byte identical whether it runs on your laptop, the cluster, or a collaborator's workstation.

In the [Docker workshop](https://nevadainbre.github.io/docker_tutorial/) we built a single image, `rnaseq-pipeline:v1`, that contains every tool our pipeline needs:

| Tool | Version |
|---|---|
| FastQC | 0.12.1 |
| fastp | 0.23.4 |
| STAR | 2.7.11a |
| featureCounts (subread) | 2.0.6 |
| MultiQC | 1.21 |

In this section we will tell Snakemake to use that image instead of per-rule Conda environments.

## Prerequisites


### Apptainer

Snakemake does not invoke Docker directly. Instead it uses [Apptainer](https://apptainer.org/) (formerly Singularity), which can pull Docker images from Docker Hub and run them without root privileges, the requirement that prevents plain Docker from being installed on most HPC clusters.

Check whether Apptainer is available on your system:



```bash
apptainer --version
```

If it isn't available and you are running locally, you can install apptainer into the same environment as your snakemake one with CONDA:

```bash
conda install apptainer
```

On most HPC clusters it is already installed. To install it on a local Linux machine, follow the [Apptainer installation guide](https://apptainer.org/docs/admin/main/installation.html). Snakemake will call `apptainer` automatically once you pass `--use-apptainer`; you never need to run Apptainer commands yourself.

!!! note "macOS / Windows"
    Apptainer is a Linux-native tool. On macOS and Windows you will need a Linux VM (e.g., via the Docker Desktop WSL2 backend or a Lima VM) to use `--use-apptainer`. For local testing on those platforms, Snakemake also supports `--use-singularity` if Singularity is installed. I have tested with WSL2 and it works. I have not been able to test with Lima VM.

### The image on Docker Hub

The image needs to be publicly accessible so Apptainer can pull it. If you built and pushed the image during the Docker workshop, your image is at:

```
docker://yourusername/rnaseq-pipeline:v1
```

If you have not pushed an image yet, you can follow along using a pre-built image we provide:

```
docker://hvasquezgross/rnaseq-pipeline:v2
```

This image is publicly available at [hub.docker.com/r/hvasquezgross/rnaseq-pipeline](https://hub.docker.com/repository/docker/hvasquezgross/rnaseq-pipeline/general).

## Updating the Snakefile

Add a global **`container:` directive** at the top of the Snakefile, immediately after the variable definitions. The `conda:` directives from the previous section can stay exactly as they are; Snakemake uses whichever mechanism you activate at runtime with `--use-conda` or `--use-apptainer`.

### Add the global container directive

Open your `Snakefile` and add the `container:` line immediately after the variable definitions:

```python hl_lines="11"
configfile: "config.yaml"

READS_DIR   = config["reads_dir"]
READS       = config["reads"]
EXT         = config["extension"]
FASTA       = config["fasta"]
GENOME_DIR  = config["genome_dir"]
GTF         = config["gtf"]
INDEX_EXTRA = config["index_extra"]

SAMPLES = sorted(glob_wildcards(READS_DIR + "/{sample}_" + READS[0] + "." + EXT).sample)

container: "docker://yourusername/rnaseq-pipeline:v1" # (1)!
```

1. Replace `yourusername` with your Docker Hub username. The `docker://` prefix tells Apptainer to pull directly from Docker Hub, with no local `docker pull` required.

!!! tip "Both directives can coexist"
    A global `container:` and per-rule `conda:` directives are **not mutually exclusive**. Leave the `conda:` lines in each rule untouched. When you pass `--use-conda`, Snakemake activates the per-rule Conda environments. When you pass `--use-apptainer`, Snakemake runs every rule inside the container instead (same Snakefile, different flag).

## Running the Pipeline

Pass `--use-apptainer` in place of `--use-conda`:

```bash
snakemake --cores 8 --use-apptainer
```

On the first run Snakemake (via Apptainer) pulls the image from Docker Hub and converts it to a local Apptainer `.sif` cache file stored in `.snakemake/singularity/`. This pull takes a minute or two the first time; subsequent runs reuse the cached `.sif` instantly.

Watch for the image pull message in the Snakemake log:

```
Pulling singularity image docker://yourusername/rnaseq-pipeline:v1.
```

After that, every rule executes inside the container exactly as it would with Conda: input and output paths are automatically bind-mounted, wildcards work identically, and `--rerun-incomplete` / `--dry-run` behave the same way.

## Combining with SLURM

`--use-apptainer` composes with the SLURM profile from the [HPC Integration](hpc.md) section:

```bash
snakemake --profile slurm --use-apptainer
```

Each SLURM job will load and run inside the container automatically. The image is pulled once to the Apptainer cache and every subsequent job reuses it, with no repeated downloads per job.

## Complete Final Snakefile

??? note "Click to expand the complete Snakefile with container directive"

    ```python
    configfile: "config.yaml"

    READS_DIR   = config["reads_dir"]
    READS       = config["reads"]
    EXT         = config["extension"]
    FASTA       = config["fasta"]
    GENOME_DIR  = config["genome_dir"]
    GTF         = config["gtf"]
    INDEX_EXTRA = config["index_extra"]

    SAMPLES = sorted(glob_wildcards(READS_DIR + "/{sample}_" + READS[0] + "." + EXT).sample)

    container: "docker://hvasquezgross/rnaseq-pipeline:v2"

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
            STAR \
                --runMode genomeGenerate \
                --runThreadN {threads} \
                --genomeDir {output} \
                --genomeFastaFiles {input.fasta} \
                --sjdbGTFfile {input.gtf} \
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

## Conda vs. Docker — When to Use Each

| | `--use-conda` | `--use-apptainer` |
|---|---|---|
| **Isolation level** | User-space only | Full OS + user-space |
| **Reproducibility** | High (pinned packages) | Very high (pinned OS layer too) |
| **Setup required** | Conda/Mamba on host | Apptainer on host |
| **HPC availability** | Depends on cluster policy | Available on most HPC clusters |
| **Disk usage** | Conda envs (~few GB) | `.sif` cache (~1–2 GB per image) |
| **Best for** | Day-to-day development | Manuscripts, shared cluster runs |

For most research computing workflows, both approaches are equally valid. The choice often comes down to which is already available on your cluster and which you are most comfortable maintaining.

## Visualising the Complete Pipeline

Now that the pipeline is fully built, generate a DAG that shows every rule and how they connect:

```bash
snakemake --dag dot | dot -Tsvg > dag_final.svg
```

Open `dag_final.svg` in a browser or image viewer. You should see a graph with this structure:

```
 star_index
     |
     +-----------+
     |           |
  fastqc      fastp       (x6 samples, x2 reads each for fastqc)
                |
           star_align      (x6 samples)
                |
         featurecounts     (all 6 BAMs together)
                |
             multiqc
```

Compare this to the single-rule DAG you generated in [Snakemake Fundamentals](fundamentals.md). The pipeline has grown from one rule producing two files to six rules, 43 jobs, automatic sample discovery from the config, and a choice of Conda or container execution, all driven by the same core Snakemake logic.

!!! note "Where we are"
    Your pipeline is complete. Head to the [Workshop Wrap-up](wrapup.md) for a summary of what you have built and pointers for applying these skills to your own research.
