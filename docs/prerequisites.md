# Course Prerequisites

## Unix Command Line

A foundational understanding of the UNIX command line is required for this workshop. Snakemake is driven entirely from the terminal, and troubleshooting pipeline failures often means navigating log files and directory structures in the shell.

Before beginning this workshop, you should be comfortable with:

* **Navigating the filesystem:** `cd`, `ls`, `pwd`
* **File manipulation:** `cp`, `mv`, `rm`, `mkdir`
* **Viewing and editing files:** `cat`, `less`, `nano` or `vim`
* **Basic stream processing:** Piping output (`|`) and redirecting to files (`>`)
* **Wildcards and globbing:** Understanding how `*` and `?` match filenames

If you need a refresher on these concepts, please review our [Introductory Unix](https://nevadainbre.github.io/unix_tutorial/latest/) materials before proceeding.

## Basic Python

Snakemake rules are embedded in a Python-like syntax, and more advanced workflows use Python expressions directly inside rule definitions. You do not need to be an expert programmer, but you should be familiar with:

* **Variables and data types:** strings, lists, and dictionaries
* **f-strings and string formatting:** e.g., `f"results/{sample}.bam"`
* **Basic control flow:** `if/else` logic and list comprehensions
* **Reading error messages:** being able to identify which line of a script caused a failure

If Python is new to you, the [Software Carpentry Python Novice lesson](https://swcarpentry.github.io/python-novice-inflammation/) is an excellent free resource.

## Conda / Mamba

This workshop uses **Conda** (via Mamba) to install Snakemake and manage per-rule software environments. You should have Miniforge (or Anaconda/Miniconda) installed on your machine before the workshop begins. Installation instructions are covered in the [Software Installation](installation.md) page.

## Recommended Background

The workshop examples are built around an RNA-seq pipeline because bioinformatics is the domain expertise of the instructors. Every Snakemake concept covered here (rules, wildcards, resource allocation, and environment management) applies equally to any field that requires multi-step computational pipelines: ecology, chemistry, climate science, image analysis, and beyond.

If bioinformatics is new to you, the RNA-seq pipeline is simply a concrete example to learn Snakemake alongside. Expand the section below for a brief overview of the file types and steps involved so you can follow along without prior bioinformatics experience.

??? info "RNA-seq pipeline background (optional)"
    * **FASTQ** files contain raw sequencing reads: short stretches of DNA sequence along with a quality score for each base.
    * **BAM** files store reads after they have been aligned to a reference genome, indicating where in the genome each read came from.
    * **Count matrices** summarize how many reads mapped to each gene, which is the starting point for differential expression analysis.
    * The general steps of the pipeline are: Quality Control → Trimming → Alignment → Quantification → Aggregated QC Report.

