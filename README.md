This repository contains a (re-)analysis of SARS-CoV-2 FASTQs from [PRJNA698267](https://www.ncbi.nlm.nih.gov/bioproject/PRJNA698267) ([Wang *et al*., Genome Medicine 2021](https://doi.org/10.1186/s13073-021-00847-5)) using the SARS-CoV-2 analysis pipeline developed for UC San Diego's Return to Learn program ([C-VIEW](https://github.com/ucsd-ccbb/C-VIEW)).

# 0: Input Data
The SRA accession list from [PRJNA698267](https://www.ncbi.nlm.nih.gov/bioproject/PRJNA698267) can be found in [`SraAccList.txt`](data/fastq/SraAccList.txt).

# 1: Mapping Reads + Sorting BAM
I'm using [Minimap2 v2.24](https://github.com/lh3/minimap2/releases/tag/v2.24) to map reads against the [NC_045512.2](data/ref/NC_045512.2.fas) reference genome, piped to [samtools v1.14](https://github.com/samtools/samtools/releases/tag/1.14) to sort and output as BAM.

## 1.1: Index Reference Genome
```bash
minimap2 -t 2 -d data/ref/NC_045512.2.fas.mmi data/ref/NC_045512.2.fas > data/ref/NC_045512.2.fas.mmi.log 2>&1
```
* **Input:** [`NC_045512.2.fas`](data/ref/NC_045512.2.fas)
* **Output:** [`NC_045512.2.fas.mmi`](data/ref/NC_045512.2.fas.mmi)
* **Log:** [`NC_045512.2.fas.mmi.log`](data/ref/NC_045512.2.fas.mmi.log)

## 1.2: Map Reads
Some of the FASTQ files are large, so rather than downloading them locally, I will map reads on-the-fly *while* downloading. The individual Minimap2 command is as follows:

```bash
minimap2 -t THREADS -a -x sr REF.MMI READ1.FASTQ.GZ READ2.FASTQ.GZ | samtools sort --threads THREADS -o SORTED.BAM
```
* `-a` means "output in the SAM format"
* `-x sr` means "use Minimap2's genomic short-read mapping preset"
  * From Wang *et al*.: "sequenced on the MGISEQ-2000 platform to generate data of **100-bp** paired-end reads"
