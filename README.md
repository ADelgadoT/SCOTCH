# SCOTCH
Single-Cell Omics for Transcriptome CHaracterization (SCOTCH): isoform-level characterization of gene expression through long-read single-cell RNA sequencing. https://www.biorxiv.org/content/10.1101/2024.04.29.590597v1

## Background
Recent development involving long-read single-cell transcriptome sequencing (lr-scRNA-Seq) represents a significant leap forward in single-cell genomics. With the recent introduction of R10 flowcells by Oxford Nanopore, computational methods should now shift focus on harnessing the unique benefits of long reads to analyze transcriptome complexity. In this context, we introduce a comprehensive suite of computational methods named Single-Cell Omics for Transcriptome CHaracterization (SCOTCH). Our method is compatible with the single-cell library preparation platform from both 10X Genomics and Parse Biosciences, facilitating the analysis of special cell populations, such as neurons, hepatocytes and developing cardiomyocytes. 

SCOTCH provides a preprocessing pipeline and a statistical pipeline. The preprocessing pipeline takes BAM files with tagged barcodes generated by vendor-supplied pipelines (wf-single-cell and parse) as input to align reads to known and novel isoforms, and outputs count matrix on both gene and transcript levels. The statistical pipeline facilitates analysis of differential transcript usage (DTU), defined by relative transcript abundances of the same gene.



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
### Step1: prepare annotation file

This step will generate annotation files for reference genome and tagged bam files. Set `--workers` argument for parallel computing. Set `--bam` argument as path to the folder saving separated bam files or path to a single bam file.

```
bamfolder="path/to/tagged/bamfile"
outputfolder="path/to/output"
reference="path/to/genome/reference/annotation.gtf"

prepare.sh -t annotation -o $outputfolder --bam $bamfolder --reference $reference --workers 30
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
Below is sample codes for DTU analysis.

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






