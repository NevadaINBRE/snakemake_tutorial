# Welcome to Nevada Bioinformatics Center - Snakemake Workshop

<img src="https://snakemake.readthedocs.io/en/stable/_static/logo-snake.svg" alt="Snakemake Logo" width="250" style="margin-bottom: 20px;">

Welcome! This hands-on workshop is designed specifically for graduate students and research scientists who want to bring structure, scalability, and reproducibility to their bioinformatics analyses.

As our field moves toward increasingly complex multi-step pipelines, managing the order of operations, re-running only what has changed, and scaling to HPC clusters by hand becomes error-prone and time-consuming. [Snakemake](https://snakemake.github.io) is the industry-standard solution for this problem.

## Why Snakemake?
Have you ever run a 12-step RNA-seq pipeline only to realize midway through that an early parameter was wrong, and had no easy way to re-run just the affected steps? Or manually submitted dozens of SLURM jobs while trying to track which samples finished and which failed?

Snakemake solves these problems by letting you define your pipeline as a set of **rules**: each specifying inputs, outputs, and the shell command or script that transforms one into the other. From that simple description, Snakemake automatically:

- Determines the order in which steps must run.
- Skips steps whose outputs are already up to date.
- Parallelizes independent steps across cores or cluster nodes.

## What You Will Learn
In this workshop, you will follow along in your own terminal to master:

1. **Snakemake Fundamentals:** Writing rules with inputs, outputs, and shell commands.
2. **Wildcards & Generalization:** Using wildcards to apply a single rule to many samples automatically.
3. **Resource Management:** Specifying threads and memory so Snakemake can schedule jobs intelligently.
4. **HPC Integration:** Submitting workflow jobs to a SLURM cluster with a single command.
5. **Reproducibility:** Pinning software environments with Conda so your pipeline runs identically on any machine.

Let's get started by ensuring your system meets the [Prerequisites](prerequisites.md).
