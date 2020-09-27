# Protocol for processing 16S sequences in QIIME2

This document describes our procedure for processing 16S amplicon libraries using the 515F-Y/926R primer set ([Parada et al. 2015](https://doi.org/10.1111/1462-2920.13023)). Amplicon sequence variants are generated with DADA2 ([Callahan et al. 2016](https://doi.org/10.1038/nmeth.3869)). For annotation, we primarily use the SILVA database but supplement with PhytoREF for plastid sequences. 

QIIME2 visualizations can be viewed [here](https://view.qiime2.org/).

We normally but not exclusively generate these libraries from DNA and RNA extracted from Sterivex filters. See our automated extraction protocols here:
* [Sterivex DNA extraction](https://dx.doi.org/10.17504/protocols.io.bc2hiyb6)
* [Sterivex RNA extraction](https://dx.doi.org/10.17504/protocols.io.bd9ti96n)

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

Remove the primers from reads with cutadapt.

* [QIIME2 cutadapt trim-paired documentation](https://docs.qiime2.org/2019.10/plugins/available/cutadapt/trim-paired/)
* [Cutadapt standalone documentation](https://cutadapt.readthedocs.io/en/stable/guide.html) ([Martin 2011](https://doi.org/10.14806/ej.17.1.200))

Parameter notes:
* This example shows trimming [linked adapters](https://cutadapt.readthedocs.io/en/stable/guide.html#linked-adapters-combined-5-and-3-adapter) as the amplicon is "framed" by both a 5' and 3' adapter. This is equivalent to usine the -a and -A options in cutadapt and shows the forward primer linked to the reverse complement of the reverse primer and vice versa. The --p-front-f and --p-front-r options may also be used.
* Number of CPU cores (--p-cores) can be increased up to the number of available cores.

```
qiime cutadapt trim-paired \
    --i-demultiplexed-sequences paired-end-demux.qza  \
    --p-cores 1 \
    --p-adapter-f ^GTGYCAGCMGCCGCGGTAA...AAACTYAAAKRAATTGRCGG \
    --p-adapter-r ^CCGYCAATTYMTTTRAGTTT...TTACCGCGGCKGCTGRCAC \
    --p-error-rate 0.1 \
    --p-overlap 3 \
    --verbose \
    --o-trimmed-sequences paired-end-demux-trimmed.qza
```

Visualize trimmed sequences:

```
qiime demux summarize \
    --i-data paired-end-demux-trimmed.qza \
    --p-n 100000 \
    --o-visualization paired-end-demux-trimmed.qzv
```

## Denoise with DADA2

Generate and quantify amplicon sequence variants ASVs with DADA2

* [QIIME2 DADA2 denoise-paired docmentation](https://docs.qiime2.org/2019.10/plugins/available/dada2/denoise-paired/)
* DADA2 [website](https://benjjneb.github.io/dada2/index.html) and [FAQ](https://benjjneb.github.io/dada2/faq.html) ([Callahan et al. 2016](https://doi.org/10.1038/nmeth.3869))

Parameter notes:
* --p-n-threads is set to 0 which uses all available cores. Adjust accordingly.
* --p-max-ee-r is set to 4 (default: 2) which we often do for MiSeq 2x300 runs. You may want to adjust the max-ee paramters (number of expected errors) depending on your data.
* --p-trunc-len-f and --p-trunc-len-r are base on typical read quality profiles we observed with MiSeq 2x300 sequencing. It is highly likely you should adjust these parameters for your own sequencing run. However, DADA2 requires a minimum of 20 nucleotides of overlap.
* If you have a large project that spans multiple sequence runs, run dada2 separately on each run. This is because different runs can have different error profiles (See https://benjjneb.github.io/dada2/bigdata.html). Since ASVs have single nucleotide level resolution, the data can later be merged (see instructions below). If merging data, ensure that your dada2 parameters are consistent. 

```
qiime dada2 denoise-paired \
    --i-demultiplexed-seqs paired-end-demux-trimmed.qza \
    --p-n-threads 0 \
    --p-trunc-q 2 \
    --p-trunc-len-f 219 \
    --p-trunc-len-r 194 \
    --p-max-ee-f 2 \
    --p-max-ee-r 4 \
    --p-n-reads-learn 1000000 \
    --p-chimera-method pooled \
    --output-dir ./asvs/ \
    --o-table table-dada2.qza \
    --o-representative-sequences rep-seqs-dada2.qza \
    --o-denoising-stats stats-dada2.qza

# Files don't actually get put in output-dir
mv table-dada2.qza ./asvs/
mv rep-seqs-dada2.qza ./asvs/
mv stats-dada2.qza ./asvs/
```

Generate and examine the DADA2 stats. You should be retaining most (>50%) of your sequences. If you are losing a large number of sequences at a certain DADA2 step, you will need to troubleshoot and adjust your DADA2 parameters accordingly.

```
qiime metadata tabulate \
  --m-input-file ./asvs/stats-dada2.qza \
  --o-visualization ./asvs/stats-dada2.qzv
```

## Merge and summarize denoised data

If you have multiple sequencing runs, proceed with merging the table and sequences from separate dada2 runs. If not, proceed to the summarization and export steps.

Merge (add additional lines for --i-tables and --i-data as needed):

```
qiime feature-table merge \
  --i-tables ./asvs/run1_table-dada2.qza \
  --i-tables ./asvs/run2_table-dada2.qza \
  --o-merged-table merged_table-dada2.qza

qiime feature-table merge-seqs \
  --i-data ./asvs/run1_rep-seqs-dada2.qza \
  --i-data ./asvs/run2_rep-seqs-dada2.qza \
  --o-merged-data merged_rep-seqs.qza
```

Summarize and export:

```
qiime tools export \
  --input-path merged_table-dada2.qza \
  --output-path asv_table

biom convert -i asv_table/feature-table.biom -o asv_table/asv-table.tsv --to-tsv
```

```
qiime tools export \
  --input-path merged_rep-seqs.qza \
  --output-path asvs

qiime feature-table tabulate-seqs \
  --i-data merged_rep-seqs.qza \
  --o-visualization merged_rep-seqs.qzv
```

If you have metadata, you can include it here:

```
qiime feature-table summarize \
  --i-table merged_table-dada2.qza \
  --o-visualization merged_table-dada2.qzv \
  --m-sample-metadata-file metadata.tsv
```

## Taxonomic annotation

* The [Naive Bayes Classifier](https://docs.qiime2.org/2019.10/tutorials/feature-classifier/) is used here.
* We place the relavent database files under a folder called 'db'

### Silva

QIIME2 compatable files can be obtained [here](https://docs.qiime2.org/2020.8/data-resources/?highlight=silva).

Extract the region between the primers and and train the classifier:

```
qiime feature-classifier extract-reads \
  --i-sequences ./db/silva/silva-138-99-seqs.qza \
  --p-f-primer GTGYCAGCMGCCGCGGTAA \
  --p-r-primer CCGYCAATTYMTTTRAGTTT \
  --o-reads ./db/silva/silva-138-99-extracts.qza

qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads ./db/silva/silva-138-99-extracts.qza \
  --i-reference-taxonomy ./db/silva/silva-138-99-tax.qza \
  --o-classifier ./db/silva/silva-138-99-classifier.qza
```

Classify the ASVs. The option --p-n-jobs -1 uses the all CPUs. Adjust accordingly. :

```
qiime feature-classifier classify-sklearn \
  --p-n-jobs -1 \
  --i-classifier ./db/silva/silva-138-99-classifier.qza \
  --i-reads ./asvs/merged_rep-seqs.qza \
  --o-classification silva_tax_sklearn.qza
  
qiime tools export \
  --input-path silva_tax_sklearn.qza \
  --output-path asv_tax_dir

mv asv_tax_dir/taxonomy.tsv asv_tax_dir/silva_taxonomy.tsv
```

### Phytoref

## Final output table
