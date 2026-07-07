# Data Setup

Before working through the workshop exercises, you need to download the sequencing data, reference genome, and build the STAR genome index. This page walks through those steps with a single setup script.

## Overview

We will use six paired-end RNA-seq samples from a 2025 *Saccharomyces cerevisiae* study investigating whether phosphorylation of the Arp8 subunit of the INO80 chromatin remodeler complex by Dbf4-Dependent Kinase (DDK) influences genome-wide transcription. Gene expression profiles from isogenic **Arp8 wild-type** and **arp8-P** (phospho-mutant: S65A/S233A) strains are compared across three biological replicates per condition. The dataset is available from [BioProject PRJNA1185750](https://www.ncbi.nlm.nih.gov/bioproject/PRJNA1185750) and published in [*Nature Communications* (2026)](https://www.nature.com/articles/s41467-026-70698-4).

The yeast genome is approximately 12 MB, small enough for the STAR index to build in minutes and for alignment to finish in seconds per sample, making it ideal for a classroom setting.

| Sample name | Condition | SRA accession |
|-------------|-----------|---------------|
| `wt_rep1`  | Arp8 wild-type replicate 1 | SRR31334598 |
| `wt_rep2`  | Arp8 wild-type replicate 2 | SRR31334596 |
| `wt_rep3`  | Arp8 wild-type replicate 3 | SRR31334594 |
| `mut_rep1` | arp8-P mutant replicate 1  | SRR31334597 |
| `mut_rep2` | arp8-P mutant replicate 2  | SRR31334595 |
| `mut_rep3` | arp8-P mutant replicate 3  | SRR31334593 |

## Expected Directory Structure

By the end of this page, your project directory should look like this:

```
snakemake_tutorial/
├── data/
│   ├── wt_rep1_R1.fastq.gz
│   ├── wt_rep1_R2.fastq.gz
│   ├── wt_rep2_R1.fastq.gz
│   ├── wt_rep2_R2.fastq.gz
│   ├── wt_rep3_R1.fastq.gz
│   ├── wt_rep3_R2.fastq.gz
│   ├── mut_rep1_R1.fastq.gz
│   ├── mut_rep1_R2.fastq.gz
│   ├── mut_rep2_R1.fastq.gz
│   ├── mut_rep2_R2.fastq.gz
│   ├── mut_rep3_R1.fastq.gz
│   └── mut_rep3_R2.fastq.gz
├── genome/
│   ├── Saccharomyces_cerevisiae.R64-1-1.dna.fa
│   ├── Saccharomyces_cerevisiae.R64-1-1.gtf
│   └── star_index/
│       └── (STAR index files)
└── Snakefile
```

## Step 1: Install Setup Tools

Activate your `snakemake` conda environment and install the tools needed to download and prepare the data:

```bash
conda activate snakemake_workshop
mamba install -c bioconda -c conda-forge sra-tools
```

## Step 2: Run the Setup Script

Save the following as `setup.sh` in your project directory, then run it. The script downloads the full FASTQ files from SRA, downloads the *S. cerevisiae* R64 reference genome and GTF from Ensembl, and builds the STAR genome index.

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "==> Creating project directories..."
mkdir -p data genome/star_index results

# ─── Download FASTQ files from SRA ───────────────────────────────────────────
declare -A SAMPLES=(
    ["wt_rep1"]="SRR31334598"
    ["wt_rep2"]="SRR31334596"
    ["wt_rep3"]="SRR31334594"
    ["mut_rep1"]="SRR31334597"
    ["mut_rep2"]="SRR31334595"
    ["mut_rep3"]="SRR31334593"
)

echo "==> Downloading FASTQ files from SRA (this may take several minutes per sample)..."
for NAME in "${!SAMPLES[@]}"; do
    ACC="${SAMPLES[$NAME]}"
    echo "    Fetching ${ACC} -> ${NAME}..."
    fasterq-dump "${ACC}" \
        --outdir data/ \
        --split-files \
        --threads 4 \
        --progress

    # Compress and rename to workshop convention
    gzip -c "data/${ACC}_1.fastq" > "data/${NAME}_R1.fastq.gz"
    gzip -c "data/${ACC}_2.fastq" > "data/${NAME}_R2.fastq.gz"

    # Remove uncompressed SRA downloads
    rm -f "data/${ACC}_1.fastq" "data/${ACC}_2.fastq"
    echo "    Done: ${NAME}"
done

# ─── Download Reference Genome and Annotation ────────────────────────────────
ENSEMBL="https://ftp.ensembl.org/pub/release-110"

echo "==> Downloading S. cerevisiae reference genome..."
wget -q --show-progress \
    -O genome/Saccharomyces_cerevisiae.R64-1-1.dna.fa.gz \
    "${ENSEMBL}/fasta/saccharomyces_cerevisiae/dna/Saccharomyces_cerevisiae.R64-1-1.dna.toplevel.fa.gz"
gunzip genome/Saccharomyces_cerevisiae.R64-1-1.dna.fa.gz

echo "==> Downloading GTF annotation..."
wget -q --show-progress \
    -O genome/Saccharomyces_cerevisiae.R64-1-1.gtf.gz \
    "${ENSEMBL}/gtf/saccharomyces_cerevisiae/Saccharomyces_cerevisiae.R64-1-1.110.gtf.gz"
gunzip genome/Saccharomyces_cerevisiae.R64-1-1.gtf.gz

# ─── Build STAR Genome Index ─────────────────────────────────────────────────
echo "==> Building STAR genome index (~3 minutes)..."
STAR \
    --runMode genomeGenerate \
    --runThreadN 4 \
    --genomeDir genome/star_index/ \
    --genomeFastaFiles genome/Saccharomyces_cerevisiae.R64-1-1.dna.fa \
    --sjdbGTFfile genome/Saccharomyces_cerevisiae.R64-1-1.gtf \
    --genomeSAindexNbases 10

echo ""
echo "==> Setup complete! Verifying files..."
echo "--- data/ ---"
ls -lh data/
echo "--- genome/ ---"
ls -lh genome/
echo ""
echo "You are ready to begin the workshop!"
```

Run it:

```bash
bash setup.sh
```

!!! note "Expected runtime and disk usage"
    | Step | Time | Space |
    |------|------|-------|
    | SRA download (6 samples) | 15–20 min | ~4 GB temporary |
    | FASTQ files (12 files) | — | ~1.5 GB (100–144 MB each) |
    | Reference genome + GTF | <1 min | ~22 MB |
    | STAR genome index | <1 min | ~150 MB |

    **Total setup time: ~15 minutes | Total space after cleanup: ~1.7 GB**

!!! tip "Why `--genomeSAindexNbases 10`?"
    STAR's default suffix array parameter (`14`) is tuned for large genomes like human (~3 GB). For the *S. cerevisiae* genome (~12 MB), a value of `10` is required, otherwise STAR prints a warning and produces a suboptimal index.

## Step 3: Verify Your Setup

Run the following to confirm all files are in place before starting the workshop:

```bash
# Should list 12 .fastq.gz files
ls data/

# Should list the .fa file, .gtf file, and star_index/
ls genome/

# Should contain SA, Genome, and other STAR index files
ls genome/star_index/
```

Once everything looks correct, open your project directory in your terminal and proceed to [Snakemake Fundamentals](fundamentals.md).
