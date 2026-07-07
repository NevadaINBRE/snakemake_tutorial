# Workshop Wrap-up

Congratulations! You have successfully built a complete, reproducible RNA-seq pipeline using Snakemake, starting from raw *Saccharomyces cerevisiae* FASTQ files and finishing with a multi-sample count matrix and an interactive MultiQC quality report.

Transitioning from ad-hoc shell scripts to a rule-based workflow manager is one of the most impactful changes you can make as a computational researcher. What began as a single FastQC rule is now a fully parallelised, cluster-ready, version-locked pipeline that handles any number of samples by changing one list.

## Cleaning Up

Once you are done exploring, you can free up disk space by removing the workshop data. Choose the level of cleanup that suits you:

```bash
# Remove all pipeline outputs (safe to regenerate any time)
rm -rf results/

# Remove downloaded FASTQ files (~1.5 GB)
rm -rf data/

# Remove the genome and STAR index (~170 MB)
rm -rf genome/

# Remove cached Conda environments and the Apptainer .sif image
rm -rf .snakemake/conda/
rm -rf .snakemake/singularity/

# Or remove the entire project directory at once
rm -rf ~/snakemake_workshop
```

!!! warning "Double-check before deleting"
    `rm -rf` is permanent. Make sure you are in (or referencing) the right directory before running it.

## What You Have Achieved

Over the course of this workshop, you have learned how to:

* **Write Snakemake Rules:** Define pipeline steps with `input:`, `output:`, and `shell:` blocks, and understand how Snakemake traces dependencies backward from a target file to determine exactly what needs to run.
* **Generalise with Wildcards:** Replace hardcoded filenames with `{sample}` wildcards and use `expand()` to apply a single rule across all samples automatically, so adding a new sample means changing one line.
* **Manage Computational Resources:** Declare `threads:` and `resources:` per rule so Snakemake parallelises jobs intelligently without overloading your machine.
* **Scale to HPC:** Submit the exact same Snakefile to a SLURM cluster using `snakemake-executor-plugin-slurm`, with `threads:` and `mem_mb` automatically translated into `#SBATCH` directives.
* **Ensure Reproducibility:** Pin software environments per rule with `conda:` directives and version-locked YAML files, so your pipeline produces the same results on any machine, today, next year, and when a reviewer asks you to revisit your analysis.

## Why This Matters

The scientific computing landscape is shifting. Journals and granting agencies increasingly mandate strict computational reproducibility. By expressing your analysis as a Snakemake workflow, you are no longer sharing a script that *might* work on someone else's machine. You are sharing a complete, self-documenting pipeline that:

- **Skips** steps whose outputs are already up to date.
- **Parallelises** automatically across whatever hardware is available.
- **Installs** the exact tools and versions used to generate your published results.
- **Scales** from a laptop to a 1000-core HPC cluster by changing a single flag.

## Next Steps & Resources

As you begin applying Snakemake to your own research, keep these resources bookmarked:

* [Snakemake Official Documentation](https://snakemake.readthedocs.io/) — The definitive reference for rule syntax, directives, configuration files, and advanced features like checkpoints and report generation.
* [Snakemake Wrappers](https://snakemake-wrappers.readthedocs.io/) — A community library of ready-to-use wrappers for hundreds of bioinformatics tools (STAR, fastp, featureCounts, and many more). Use a wrapper and the `conda:` environment is already written for you.
* [Snakemake Workflows](https://github.com/snakemake-workflows) — A curated collection of peer-reviewed, production-grade Snakemake pipelines for RNA-seq, ChIP-seq, variant calling, and more. A great starting point for real analyses.

Thank you for taking the time to learn Snakemake with us. We look forward to seeing the reproducible, scalable science you publish next!

*— Hans and the Nevada Bioinformatics Center Team*

