# Software Installation

Before the workshop begins, you will need a working Conda/Mamba installation and a Snakemake environment on your local machine.

## 1. Install Miniforge (Conda + Mamba)

We recommend **Miniforge**, a minimal Conda installer that ships with the fast `mamba` solver pre-configured and defaults to the community `conda-forge` channel. If you already have Anaconda or Miniconda installed and `mamba` is available, you can skip this step.

* Navigate to the [Miniforge releases page](https://github.com/conda-forge/miniforge/releases/latest).
* Download and run the installer for your operating system.

=== "Windows"
    Download the **Miniforge3 Windows x86_64** installer (`.exe`). Run it and follow the prompts. When asked, select **"Add Miniforge3 to my PATH"** so that `conda` and `mamba` are available from your terminal.

    We strongly recommend using the **Windows Subsystem for Linux (WSL 2)** terminal for this workshop, as Snakemake's cluster integration is designed for Linux environments. Once WSL is set up, install the Linux version of Miniforge inside your WSL terminal.

=== "macOS"
    Download the installer matching your chip:

    * **Apple Silicon (M-series):** `Miniforge3-MacOSX-arm64.sh`
    * **Intel:** `Miniforge3-MacOSX-x86_64.sh`

    Run it from your terminal:
    ```bash
    bash Miniforge3-MacOSX-*.sh
    ```
    Follow the prompts and allow the installer to initialize Conda in your shell.

=== "Linux"
    Download and run the Linux x86_64 installer:
    ```bash
    wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
    bash Miniforge3-Linux-x86_64.sh
    ```
    Follow the prompts and allow the installer to initialize Conda.

### Verify Your Conda Installation

Close and reopen your terminal, then run:

```bash
conda --version
mamba --version
```

You should see version numbers printed for both commands.

## 2. Create a Snakemake Conda Environment

Rather than installing Snakemake into your base environment, we will create a dedicated environment. This is best practice and avoids dependency conflicts.

This command also pre-installs all the bioinformatics tools used in the workshop examples (FastQC, fastp, STAR, featureCounts, MultiQC). This means every command in the workshop will work right away. Later, in the [Reproducibility](reproducibility.md) section, you will see how to assign each tool its own isolated, version-locked environment per rule, and why that is a better long-term approach than a single shared environment.

```bash
mamba create -n snakemake_workshop -c conda-forge -c bioconda \
    snakemake \
    fastqc=0.12.1 \
    fastp=0.23.4 \
    star=2.7.11a \
    subread=2.0.6 \
    multiqc=1.21
```

!!! note "This may take 5–10 minutes"
    Conda needs to solve and download several large packages, including STAR. Grab a coffee.

```bash
conda activate snakemake_workshop
```

### Verify Your Snakemake Installation

```bash
snakemake --version
```

You should see the Snakemake version number (e.g., `8.x.x` or `9.x.x`). If so, you are ready for the workshop!


!!! tip "Activate your environment first"
    Remember to run `conda activate snakemake_workshop` at the start of each new terminal session before working through the exercises.
