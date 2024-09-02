# SCOTCH 

Single-Cell Omics for Transcriptome CHaracterization (SCOTCH): isoform-level characterization of gene expression through long-read single-cell RNA sequencing. See our pre-print at [here](https://www.biorxiv.org/content/10.1101/2024.04.29.590597v1). All codes and analysis results in the manuscript can be found at [here](https://github.com/WGLab/SCOTCH_reproduction)

## Background
Recent development involving long-read single-cell transcriptome sequencing (lr-scRNA-Seq) represents a significant leap forward in single-cell genomics. With the recent introduction of R10 flowcells by Oxford Nanopore, computational methods should now shift focus on harnessing the unique benefits of long reads to analyze transcriptome complexity. In this context, we introduce a comprehensive suite of computational methods named Single-Cell Omics for Transcriptome CHaracterization (SCOTCH). Our method is compatible with the single-cell library preparation platform from 10X Genomics, Pacbio Biosciences, and Parse Biosciences, facilitating the analysis of special cell populations, such as neurons, hepatocytes and developing cardiomyocytes. 

SCOTCH offers both a preprocessing and a statistical pipeline. The preprocessing pipeline accepts BAM files with tagged barcodes generated by vendor-supplied pipelines (e.g., wf-single-cell) and aligns reads to both known and novel isoforms. SCOTCH can leverage existing high-quality gene annotations, update these annotations using the data to enhance novel isoform identification, or operate entirely annotation-free. SCOTCH outputs count matrices at both the gene and transcript levels. The statistical pipeline supports the analysis of differential transcript usage (DTU), which is determined by the relative abundances of transcripts within the same gene.



## Installation
To avoid package version conflicts, we strongly recommand to use conda to set up the environment. If you don't have conda installed, you can run codes below in linux to install.

```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh 
bash Miniconda3-latest-Linux-x86_64.sh
```
After installing conda, SCOTCH sources can be downloaded:
`git clone https://github.com/WGLab/SCOTCH.git`

```
cd SCOTCH
conda env create --name SCOTCH --file scotch.yml
conda activate SCOTCH
```

## Run SCOTCH preprocessing pipeline

SCOTCH accepts BAM files tagged by vendor-supplied pipelines for its preprocessing pipeline. Users can process one sample at a time or multiple samples simultaneously. When preprocessing multiple samples together, SCOTCH generates a unified gene/isoform annotation based on all BAM files. This approach is particularly beneficial in studies involving multiple samples, as it ensures consistent identification and consolidation of novel isoforms across different samples, addressing the common challenge of comparing isoforms identified independently.

### Output Directory Structure

The main function for the preprocessing pipeline is `main_preprocessing.py`. Simply assign a directory for the argument `--target` and SCOTCH will save all output files under the `target` directory. Below is the directory structure you can expect:

- **`auxiliary/`**: This folder contains information of read-isoform mappings, cell barcodes, UMI, and exon coordinates of Isoforms.

- **`bam/`**: Information of BAM files preprocessed, including read names, barcodes, read length etc.

- **`compatible_matrix/`**: This directory holds the compatibility matrices of read - isoform alignments.

- **`reference/`**: This folder includes the reference files used or generated by SCOTCH. 

- **`count_matrix/`**: This folder holds the count matrices on both gene and isoform levels generated by SCOTCH.


### Step1: prepare annotation file

In this step, SCOTCH will generate annotation files for the reference genome and tagged BAM files. SCOTCH offers three modes for generating gene annotations: 
1. **Annotation-Only Mode**: SCOTCH can rely entirely on existing gene annotations. This mode allows for the discovery of novel isoforms defined by combinations of known exons. Set `--reference` as path to gene annotation .gtf file, or use default annotation file pre-computated by SCOTCH (human hg38). In addition, set `--update_gtf_off`.
2. **Semi-Annotation Mode**: SCOTCH can use BAM files from one or multiple samples to update and refine existing gene annotations. This mode allows for the discovery of de novo (sub)exons with more types of novel isoforms than annotation-only mode. Set `--reference` as path to gene annotation .gtf file, or use default annotation file pre-computated by SCOTCH (human hg38). In addition, set `--update_gtf`.
3. **Annotation-Free Mode**: SCOTCH can generate gene and isoform annotations based solely on BAM files, allowing for the discovery of novel genes and isoforms. Set `--reference` as `None`. 

Below is an example of generating annotation in the Semi-Annotation Mode for two samples simultanuously.

`--bam`: path(s) to the folder(s) saving separated bam files or path(s) to a single/multiple bam file(s).
`--workers`: number of threads for parallel computing. 
`--coverage_threshold_exon`: coverage threshold to support exon discovery, percentage to the maximum coverage, larger values will be more conservative.
`--coverage_threshold_splicing`: threshold to support splicing discovery, percentage to the maximum splicing junctions, larger values will be more conservative.
`--z_score_threshold`: z score threshold to discovery sharp changes of read coverage, larger values will be more conservative.


```
python3 src/main_preprocessing.py \
--task annotation \
--target path/to/output/folder/of/sample1 path/to/output/folder/of/sample2 \
--bam path/to/bam/file/or/bamfolder/sample1 path/to/bam/file/or/bamfolder/sample2 \
--reference path/to/genes.gtf \
--coverage_threshold_exon 0.02 \
--coverage_threshold_splicing 0.02 \
--z_score_threshold 10 \
--workers 30
```

### Step2: generate compatible matrix

This step will generate read-isoform compatible matrix. `-o` and `--bam` are the same with step1. Set `-match` argument for the threshold of read-exon mapping percentage. For example, setting 0.2 means reads covers >80% of the exon length as mapped, and reads covers <20% of the exon length as unmapped.

```
prepare.sh -t matrix -o $outputfolder --bam $bamfolder -match 0.2 
```

To speed up the step, job arrays can be submitted in SLURM. For example:

```
#! /bin/bash -l
#SBATCH --nodes=1
#SBATCH --ntasks=1 
#SBATCH --cpus-per-task=1
#SBATCH --mem=200G
#SBATCH --array=0-99
source ~/.bashrc
conda activate SCOTCH

bamfolder="path/to/tagged/bamfile"
outputfolder="path/to/output"

prepare.sh -t matrix -o $outputfolder --bam $bamfolder -match 0.2 --job_index ${SLURM_ARRAY_TASK_ID} --jobs 100
```

### Step2: generate count matrix
This step will generate gene- and isoform-level copunt matrix. `-o` and `--bam` are the same with step1. Set '-novel_read_n' for the threshold of filtering novel isoform. Novel isoform with the number of mapped reads below this threshold will be treated as uncategorized.

```
prepare.sh -t count -o $outputfolder --novel_read_n 10 --workers 20
```

## Run SCOTCH statistical pipeline

### Installation

In R, run below codes to install SCOTCH statistical pipeline.

```
if (!requireNamespace("devtools", quietly = TRUE))
install.packages("devtools")
library("devtools")
install_github("WGLab/SCOTCH")
```

### DTU analysis
Below is sample codes for DTU analysis. Example data can be found in `/data` folder.

```
library(SCOTCH)

#----read gene-level count matrix-----#
sample8_CD4_gene=as.matrix(read.csv('gene_count_1.csv',row.names = 'X'))
sample8_CD8_gene=as.matrix(read.csv('gene_count_2.csv',row.names = 'X'))

#----read transcript-level count matrix-----#
sample8_CD4_transcript=as.matrix(read.csv("transcript_count_1.csv",row.names = 'X'))
gene_transcript_CD4_df = data.frame(genes=str_remove(colnames(sample8_CD4_transcript),"\\.(ENST|novel|uncategorized).+"),
                                    transcripts=colnames(sample8_CD4_transcript))

sample8_CD8_transcript=as.matrix(read.csv("transcript_count_2.csv",row.names = 'X'))
gene_transcript_CD8_df = data.frame(genes=str_remove(colnames(sample8_CD8_transcript),"\\.(ENST|novel|uncategorized).+"),
                                    transcripts=colnames(sample8_CD8_transcript))

#----gene-level analysis-----#
df_gene = scotch_gene(sample8_CD4_gene, sample8_CD8_gene), epsilon=0.01,ncores=10)%>%
  filter(pct1>=0.01|pct2>=0.01)

#----transcript-level analysis-----#
df_transcript = scotch_transcript(gene_transcript_CD4_df,gene_transcript_CD8_df, 
                                  sample8_CD4_transcript, sample8_CD8_transcript, ncores=10)
df_scotch = df_gene%>%left_join(df_transcript,by=c('genes'='gene'))
```

SCOTCH will output the following statistics in results:
```
genes: gene name
pct1: percent of cells expression the gene in cell population 1
pct2: percent of cells expression the gene in cell population 2
logFC: fold change in log scale
p_gene: p value on the gene level
pct_diff: difference between pct1 and pct2
p_gene_adj: adjusted p value of differential gene expression
isoforms: isoform name
alpha1: estimated parameter of the dirichlet distribution for cell population 1
alpha2: estimated parameter of the dirichlet distribution for cell population 2
TU1: relative transcript usage of cell population 1
TU2: relative transcript usage of cell population 2
TU_diff: difference of TU1 and TU2
TU_var1: variance of TU1
TU_var2: variance of TU2
dispersion1: dispersion parameter for cell population 1
dispersion2: dispersion parameter for cell population 2
isoform_switch: whether there is isoform switching event
isoform_switch_ES: effect size of isoform switching event
p_DTU_gene: p value for differential transcript usage analysis for the whole gene
p_transcript: p value for differential transcript usage analysis for the specific transcript
p_transcript_adj: adjusted p value for differential transcript usage analysis for the specific transcript
p_DTU_gene_adj: adjusted p value for differential transcript usage analysis for the whole gene
``` 



