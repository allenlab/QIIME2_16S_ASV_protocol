# Protocol for processing 16S sequences in QIIME2

This document describes our procedure for processing 16S amplicon libraries using the 515F-Y/926R primer set [Parada et al. 2015](https://doi.org/10.1111/1462-2920.13023). For annotation, we primarily use the SILVA database but supplement with PhytoREF for plastid sequences. 

## Requirements
* [QIIME2 Version 2019.10](https://docs.qiime2.org/2019.10/) or higher
* [Silva 138](https://docs.qiime2.org/2020.8/data-resources/?highlight=silva) SSURef NR99 full-length sequences and taxonomy
* [PhytoREF](http://phytoref.sb-roscoff.fr/)

## Start

Activate your conda environment for QIIME2 and cd to your working directory

```
source activate qiime2-2019.10
cd $working_directory
```

## Import files

## Trim reads

## Denoise with DADA2

## Merge and summarize denoised data

## Taxonomic annotation

## Final output table
