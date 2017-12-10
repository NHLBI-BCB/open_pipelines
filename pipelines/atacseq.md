# ATAC-seq pipeline

This repository contains a pipeline to process ATAC-seq data. It does adapter trimming, mapping, peak calling, and creates bigwig tracks, TSS enrichment files, and other outputs. You can download the latest version from the [releases page](https://github.com/databio/ATACseq/releases) and a history of version changes is in the [CHANGELOG](CHANGELOG.md).


## Installation and configuration

### Prequisites

**Python packages**. This pipeline uses [pypiper](https://github.com/epigen/pypiper) and [looper](https://github.com/epigen/looper). You can do a user-specific install of these like this:

```
pip install --user https://github.com/epigen/pypiper/zipball/v0.6
pip install --user https://github.com/epigen/looper/zipball/v0.7.2
```

**Required executables**. You will need some common bioinformatics software installed. The list is specified in the pipeline configuration file ([atacseq.yaml](atacseq.yaml)) tools section.

**Static files**. This pipeline requires static files which are specified in the [pipeline configuration file](atacseq.yaml).


## Usage

 - Clone the pipeline repository: `git clone git@github.com:epigen/open_pipelines.git`;
 - Adapt the [pipeline configuration file](atacseq.yaml) to point to specific software if needed;
 - Create a sample annotation sheet containing the variables `sample_name`, `protocol`, and `organism`;
 - Create a project configuration file that points to the [pipeline interface file](../pipeline_interface.yaml) and the sample annotation sheet;
 - Run pipelines using looper `looper run project_config.yaml`.

More detailed instructions or creating a project configuration file and sample annotation sheet canbe found in the [Looper documentation](http://looper.readthedocs.io).


## Pipeline steps

### Merging input files

If given more than one BAM file as input, the pipeline will merge them before begining processing. The merged, unmapped inpu BAM file will be output in `$sample_name/unmapped`. This file is temporary and will be removed if the pipeline finishes successfully.

### Sequencing read quality control with FASTQC

[FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) is ran on the unaligned input BAM files for quality control.

An HTML report and accompaning zip file will be output in the root directory of each sample.

### Read trimming

Reads are trimmed for adapters prior to alignment. 

Adapter sequences to be trimmed can be specified in a FASTA file which is stated in the [pipeline configuration file](atacseq.yaml) under `resources: adapters`.

Two trimming programs can be selected: [trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) and [skewer](https://github.com/relipmoc/skewer) in the [pipeline configuration file](atacseq.yaml) under `parameters: trimmer`. While rigorous benchmarking of both trimmers could be performed, the reason to use skewer is its considerable speed compared with trimmomatic and the fact that it is available as a binary executable rather than a Java jar.

These produce FASTQ files for each read pair and one file with unpaired reads, which are stored under `$sample_name/unmapped/`. These files are temporary and will be removed if the pipeline finishes sucessfully.

### Read alignment

Read alignment is done with [Bowtie2](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml). This file is then sorted and indexed with [sambamba](http://lomereiter.github.io/sambamba/). Alignment is done with the `--very-sensitive` parameters and in paired-end mode, only fragments with at most 2000bp are allowed. An aligned BAM file is produced in sample output directory `$sample_name/mapped`.

### Read filtering

Reads are filtered quite stringently, specially in paired-end mode. This fits the many clinically-oriented projects in the Bock lab but may be adjusted. 

The minimum accepted mapping quality in 30 (phred scale) and in paired-end mode only concordant proper pairs are kept. In addition, optical duplicates are removed with [sambamba](http://lomereiter.github.io/sambamba/) and only nuclear genome reads are kept.

### Peak calling

Peaks can be called with [MACS2](https://github.com/taoliu/MACS) or [spp](https://github.com/kundajelab/phantompeakqualtools) but since ATAC-seq peak calling with MACS2 works (surprisingly) very well and spp is not really a command-line program but a script, I strongly recommend using MACS2 only.

MACS2 is run with the `--no-model --extsize 147` parameters and outputs peak calls in the `$sample_name/peaks` directory in both narrowPeak and excel formats.
For genomes with annotation available, I strongly recommend filtering the peak calls using a blacklist file such as the ENCODE (https://sites.google.com/site/anshulkundaje/projects/blacklists).

### Quality control

#### bigWig file

A bigWig file with signal along the genome will be produced which can be used for visualizations in genome browsers. This requires some [UCSC tools](http://hgdownload.soe.ucsc.edu/admin/exe/linux.x86_64/) to be available, particularly `bedGraphToBigWig`.
This file can be normalized to the total library size if specified in the [pipeline configuration file](atacseq.yaml) under `parameters: normalize_tracks`. The normalization can take a normalization factor `parameters: norm_factor` under consideration which will make the scalling of the bigWig file comparable between samples.

#### Fragment distributions

Libray fragment distribution can be an indicator of library quality and refects the abundance of nucleosome-free or nucleosome fragments. **For paired-end samples** a plot of fragment distributions will be produced and stored in the sample's root directory. A mixture of distributions will try to be fit to the observed distribution: an inverse exponential distribution modeling the nucleosome-free fragments and four gaussians modeling various nucleosomal distributions.

#### FRiP

One of the most used measures of signal-to-noise ratio in an ATAC-seq library is the fraction of reads in peaks (FRiP). This is simply the fraction of filtered reads that is overlapping peaks (from the own sample) over all filtered reads. For a more complete description and reasoning behind this metric see [Landt et al 2012](https://dx.doi.org/doi:10.1101/gr.136184.111). A file holding the number of reads overlapping peaks will be output in the sample's root directory and the actual FRiP value will be reported in the statistics.

#### Cross-correlation enrichment

Two additional quality metric will be computed: the normalized strand cross-correlation (NSC) and relative strand cross-correlation (RSC). These metrics were defined as a measure of signal-to-noise ratio in ChIP-seq experiments and don't have the same meaning for ATAC-seq data (see [Landt et al 2012](https://dx.doi.org/doi:10.1101/gr.136184.111)). However, a passing quality ATAC-seq library should have a NSC higher than 1 and generally a higher RSC value is related with better quality samples.


## Collecting statistics from pipeline runs

### Statistics

 - `fastqc_GC_perc`: GC percentage of all sequenced reads from FASTQC report.
 - `fastqc_read_length`: read length as determined from FASTQC report.
 - `fastqc_total_pass_filter_reads`: number of pass filter reads from FASTQC report.
 - `fastqc_poor_quality_perc`: number of poor quality reads from FASTQC report
 - `trim_short_perc`: percentage of reads dropped because of too short length after trimming
 - `trim_empty_perc`: percentage of reads dropped because empty after trimming
 - `trim_trim_loss_perc`: percentage of reads lost during trimming
 - `trim_surviving_perc`: percentage of reads surviving after trimming
 - `trim_trimmed_perc`: percentage of reads that were trimmed
 - `trim_untrimmed_perc`: percentage of reads that were untrimmed
 - `unique_aligned_perc`: percentage of reads that aligned uniquely
 - `perc_aligned`: percentage of reads that were aligned
 - `not_aligned_perc`: percentage of reads that were not aligned
 - `multiple_aligned_perc`: percentage of reads that aligned to multiple reference locations
 - `MT_filtered_paired_ends`: number of mitochondrial paired-end reads available after filtering
 - `MT_duplicate_percentage`: percentage of mitochondrial reads that are duplicate
 - `MT_filtered_single_ends`: number of mithochondrial single-end reads available after filtering
 - `filtered_paired_ends`: number of paired-end reads available after filtering
 - `duplicate_percentage`: percentage of duplicate reads
 - `filtered_single_ends`: number of single-end reads available after filtering
 - `NSC`: normalized strand cross-correlation
 - `RSC`: relative strand cross-correlation
 - `peaks`: number of called peaks
 - `frip`: fraction of reads in peaks (FRiP)
 - `Time`: pipeline run time 
 - `Success`: time pipeline run finished


## Contributing

Pull requests welcome. Active development should occur in the development branch.

## Contributors

* Andre Rendeiro, arendeiro@cemm.oeaw.ac.at