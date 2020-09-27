# Protocol for processing 16S sequences in QIIME2

This document describes our procedure for processing 16S amplicon libraries using the 515F-Y/926R primer set ([Parada et al. 2015](https://doi.org/10.1111/1462-2920.13023)). For annotation, we primarily use the SILVA database but supplement with PhytoREF for plastid sequences. 

QIIME2 visualizations can be viewed [here](https://view.qiime2.org/).

## Requirements
* [QIIME2 Version 2019.10](https://docs.qiime2.org/2019.10/) or higher
* [Silva 138](https://docs.qiime2.org/2020.8/data-resources/?highlight=silva) SSURef NR99 full-length sequences and taxonomy
* [PhytoREF](http://phytoref.sb-roscoff.fr/)

## Important Notes
* This is a generic protocol. We often append a unique project ID to the beginning of all filenames throughout.
* Some of the specified options may not be appropriate and need to be adjusted according to your data and configuration. Consult the [QIIME2 documentation](https://docs.qiime2.org/2019.10/) and other software documentation where appropriate to fully understand what each command and option is doing. We have attempted to point out specific examples where parameters need to be adjusted but these should not be viewed as a comprehensive list.
* Refer to the QIIME2 documentation [here](https://docs.qiime2.org/2019.10/citation/) on how to properly cite QIIME2 and plugins used.

## Start

Activate your conda environment for QIIME2 and cd to your working directory

```
source activate qiime2-2019.10
cd $working_directory
```

## Import files

This assumes your data are provided as demultiplexed fastq files with PHRED 33 encoded quality scores. If your data are in a different format, see the [QIIME2 documentation on importing data](https://docs.qiime2.org/2019.10/tutorials/importing/).

First, generate a fastq manifest file which maps sample IDs to the full path of your fastq files (compressed as fastq.gz is also fine). The manifest is a tab-delimited file (.tsv) with the following headers:

```
sample-id forward-absolute-filepath reverse-absolute-filepath
```

Import the files and validate the output.

```
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path manifest.tsv \
  --output-path paired-end-demux.qza \
  --input-format PairedEndFastqManifestPhred33V2

qiime tools validate paired-end-demux.qza
```

A visualization of the imported sequences is often helpful.
```
qiime demux summarize \
  --i-data paired-end-demux.qza \
  --o-visualization paired-end-demux.qzv
```

## Trim reads

## Denoise with DADA2

## Merge and summarize denoised data

## Taxonomic annotation

## Final output table
