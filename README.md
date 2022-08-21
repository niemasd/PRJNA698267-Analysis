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

However, I'll use named pipes for the FASTQ files to download + map on-the-fly, and I'll throw out unmapped reads (via Minimap2's `--sam-hit-only` flag):

```bash
minimap2 --sam-hit-only -t THREADS -a -x sr REF.MMI <(fasterq-dump SRR_NUMBER --concatenate-reads --stdout) | samtools sort --threads THREADS -o SORTED.BAM
```

Putting everything together, here's the looped command for doing everything:

```bash
for s in $(cat data/fastq/SraAccList.txt) ; do minimap2 --sam-hit-only -t 1 -a -x sr data/ref/NC_045512.2.fas.mmi <(fasterq-dump $s --concatenate-reads --stdout) 2> "data/bam/$s.01.sorted.untrimmed.bam.log" | samtools sort --threads 1 -o "data/bam/$s.01.sorted.untrimmed.bam" ; done
```

The resulting BAM files can be found in the [`data/bam`](data/bam) folder.

# 2: Trimming BAMs + Sorting Trimmed BAMs
I'm using [iVar v1.3.1](https://github.com/andersen-lab/ivar/releases/tag/v1.3.1) to quality trim the mapped reads, and I'm using [samtools v1.14](https://github.com/samtools/samtools/releases/tag/1.14) to sort the trimmed BAMs.

```bash
ivar trim -e -i SORTED_UNTRIMMED.BAM -p UNSORTED_TRIMMED_PREFIX
samtools sort --threads 1 -o SORTED_TRIMMED.BAM UNSORTED_TRIMMED.BAM
```
* `-e` means "include reads without primers" (we're not trimming primers, as there were no primers to trim)

The resulting trimmed BAM files can be found in the [`data/trimmed_bam`](data/trimmed_bam) folder.

# 3: Calculating Coverage
I'm using [SamBamViz v0.0.9](https://github.com/niemasd/SamBamViz/releases/tag/0.0.9) to count the number of each base at each position of the genome from the trimmed BAMs. I'm using a minimum base quality score of 20 (`-q 20`) to match standard consensus calling approaches such as iVar Consensus.

```bash
SamBamViz.py -i TRIMMED_SORTED.BAM -o OUT_TSV -q 20 --start_at_one
```

The resulting SamBamViz output files can be found in the [`data/sambamviz`](data/sambamviz) folder and can be visualized in the [SamBamViz Web App](https://niema.net/SamBamViz). Folks are especially interested in (1-based) positions 8782 and 28144, so for convenience, the following table lists the base counts at these two positions for all samples:

| Sample | 8782 A | 8782 C | 8782 G | 8782 T | 28144 A | 28144 C | 28144 G | 28144 T |
| ------ | ------ | ------ | ------ | ------ | ------- | ------- | ------- | ------- |
| SRR13615939 | 11 | 980 | 25 | 13 |8 | 6 | 9 | 2103 |
| SRR13615940 | 1 | 0 | 1 | 2 |0 | 4 | 1 | 1 |
| SRR13615941 | 1184 | 57265 | 2163 | 894 |545 | 449 | 582 | 92304 |
| SRR13615942 | 73 | 6348 | 105 | 63 |24 | 21 | 18 | 11717 |
| SRR13615943 | 1316 | 105094 | 3231 | 1526 |517 | 599 | 555 | 141809 |
| SRR13615944 | 2 | 1 | 2 | 115 |0 | 130 | 1 | 4 |
| SRR13615945 | 4 | 1 | 2 | 182 |0 | 160 | 0 | 8 |
| SRR13615946 | 1 | 1 | 0 | 146 |0 | 256 | 0 | 1 |
| SRR13615947 | 5 | 6 | 6 | 235 |0 | 308 | 1 | 3 |
| SRR13615948 | 1717 | 83115 | 3368 | 1879 |830 | 542 | 815 | 143587 |
| SRR13615949 | 2499 | 104233 | 2520 | 1200 |478 | 398 | 490 | 115324 |
| SRR13615950 | 129 | 38 | 238 | 10087 |43 | 16394 | 69 | 210 |
| SRR13615951 | 20 | 1865 | 55 | 15 |2 | 7 | 10 | 3821 |
| SRR13615952 | 640 | 608 | 734 | 31937 |140 | 40991 | 181 | 1096 |
| SRR13615953 | 13 | 14 | 19 | 603 |6 | 1215 | 6 | 51 |
| SRR13615954 | 23 | 3 | 23 | 857 |2 | 1311 | 5 | 34 |
| SRR13615955 | 5702 | 194773 | 9067 | 6660 |2208 | 1620 | 2215 | 418563 |
| SRR13615956 | 3192 | 153044 | 4986 | 3190 |702 | 2482 | 679 | 193859 |
| SRR13615957 | 103 | 31 | 166 | 8330 |36 | 14857 | 37 | 197 |
| SRR13615958 | 263 | 176 | 329 | 14665 |42 | 18814 | 67 | 401 |
| SRR13615959 | 15 | 9 | 20 | 638 |8 | 961 | 4 | 33 |
| SRR13615960 | 97 | 39 | 94 | 2681 |11 | 2522 | 13 | 96 |
| SRR13615961 | 628 | 192 | 893 | 24419 |191 | 41897 | 244 | 836 |
| SRR13615962 | 230 | 16410 | 228 | 127 |67 | 52 | 65 | 23761 |
| SRR13615963 | 2380 | 816 | 3437 | 106153 |699 | 156238 | 1023 | 3061 |
| SRR13615964 | 2 | 12 | 13 | 562 |2 | 1128 | 2 | 26 |
| SRR13615965 | 90 | 26 | 138 | 4132 |26 | 6416 | 22 | 121 |
| SRR13615966 | 0 | 12 | 1 | 6 |0 | 4 | 0 | 29 |
| SRR13615967 | 12 | 864 | 38 | 19 |2 | 6 | 1 | 1640 |
| SRR13615968 | 2 | 14 | 1 | 4 |0 | 7 | 0 | 31 |
| SRR13615969 | 71 | 8420 | 206 | 124 |21 | 13 | 32 | 12017 |
| SRR13615970 | 0 | 30 | 4 | 2 |0 | 6 | 0 | 42 |
| SRR13615971 | 23 | 494 | 31 | 42 |5 | 8 | 6 | 974 |
| SRR13615972 | 1 | 45 | 1 | 4 |1 | 12 | 0 | 67 |
| SRR13615973 | 0 | 19 | 0 | 0 |0 | 0 | 0 | 25 |
| SRR13615974 | 0 | 5 | 1 | 8 |0 | 9 | 1 | 5 |
| SRR13615975 | 0 | 0 | 0 | 3 |0 | 4 | 0 | 0 |
| SRR13615976 | 219 | 14780 | 477 | 264 |69 | 78 | 85 | 20804 |
| SRR13615977 | 565 | 32215 | 1083 | 632 |160 | 155 | 217 | 40043 |
| SRR13615978 | 1286 | 119495 | 2048 | 949 |377 | 314 | 467 | 91011 |
| SRR13615979 | 0 | 12 | 0 | 3 |0 | 6 | 0 | 4 |
| SRR13615980 | 1 | 135 | 2 | 6 |0 | 11 | 0 | 237 |
| SRR13615981 | 1 | 151 | 1 | 2 |0 | 0 | 0 | 140 |
| SRR13615982 | 5 | 962 | 15 | 11 |4 | 8 | 4 | 1475 |
| SRR13615983 | 3 | 819 | 3 | 0 |0 | 1 | 0 | 774 |
| SRR13615984 | 8 | 625 | 26 | 14 |4 | 5 | 3 | 959 |
| SRR13615985 | 0 | 5 | 3 | 70 |0 | 95 | 1 | 6 |
| SRR13615986 | 1 | 9 | 2 | 34 |1 | 55 | 0 | 4 |
| SRR13615987 | 105 | 30 | 167 | 5343 |32 | 8058 | 38 | 155 |
| SRR13615988 | 2159 | 607 | 3632 | 102798 |518 | 129902 | 755 | 2679 |
| SRR13615989 | 2 | 6 | 10 | 454 |1 | 705 | 6 | 12 |
| SRR13615990 | 1127 | 621 | 2138 | 42662 |286 | 78454 | 491 | 1758 |
| SRR13615991 | 5 | 57 | 11 | 225 |3 | 518 | 4 | 146 |
| SRR13615992 | 1128 | 340 | 1902 | 28365 |323 | 55367 | 633 | 2426 |
| SRR13615993 | 335 | 183 | 495 | 9257 |228 | 15219 | 218 | 603 |
| SRR13615994 | 20 | 61 | 64 | 874 |7 | 1500 | 14 | 162 |
| SRR13615995 | 12 | 6 | 19 | 546 |3 | 915 | 2 | 16 |
| SRR13615996 | 7111 | 1938 | 10550 | 268362 |2564 | 506662 | 3549 | 10085 |
| SRR13615997 | 1657 | 422 | 1655 | 88554 |278 | 123176 | 389 | 2118 |
| SRR13615998 | 80 | 107 | 140 | 5143 |31 | 6822 | 42 | 299 |
| SRR13615999 | 2350 | 774 | 5221 | 140060 |940 | 173788 | 1295 | 4703 |
| SRR13616000 | 52 | 2067 | 60 | 123 |14 | 218 | 11 | 2184 |
| SRR13616001 | 460 | 20845 | 338 | 165 |98 | 74 | 96 | 15231 |
| SRR13616002 | 97 | 11107 | 194 | 107 |45 | 27 | 24 | 19135 |
| SRR13616003 | 2110 | 100225 | 4179 | 2322 |943 | 696 | 854 | 154071 |
| SRR13616004 | 646 | 39932 | 629 | 330 |116 | 93 | 141 | 31912 |
| SRR13616005 | 3 | 119 | 9 | 109 |2 | 184 | 5 | 269 |
| SRR13616006 | 1166 | 351 | 2295 | 48217 |291 | 82530 | 482 | 1821 |
| SRR13616007 | 4 | 229 | 11 | 61 |0 | 149 | 6 | 399 |
| SRR13616008 | 3 | 159 | 0 | 0 |0 | 0 | 0 | 157 |
| SRR13616009 | 0 | 0 | 0 | 1 |0 | 0 | 0 | 35 |
| SRR13616010 | 7 | 1 | 8 | 599 |1 | 638 | 2 | 5962 |
| SRR13616011 | 35 | 5 | 67 | 3032 |10 | 5519 | 14 | 55 |
| SRR13616012 | 516 | 166 | 1186 | 40120 |236 | 80395 | 329 | 1064 |
| SRR13616013 | 125 | 45 | 234 | 5355 |55 | 8334 | 63 | 188 |
| SRR13616014 | 253 | 56 | 226 | 12039 |51 | 10796 | 75 | 203 |
| SRR13616015 | 0 | 23 | 1 | 1 |0 | 1 | 0 | 37 |
| SRR13616016 | 158 | 6937 | 344 | 197 |65 | 66 | 105 | 16288 |
| SRR13616017 | 364 | 12278 | 700 | 387 |119 | 111 | 133 | 20317 |
| SRR13616018 | 0 | 27 | 0 | 0 |0 | 0 | 0 | 41 |
| SRR13616019 | 280 | 8145 | 436 | 328 |25 | 17 | 35 | 11024 |
| SRR13616020 | 0 | 9 | 0 | 16 |0 | 39 | 0 | 11 |
| SRR13616021 | 35 | 1801 | 92 | 38 |1 | 4 | 0 | 2939 |
| SRR13616022 | 0 | 9 | 0 | 0 |0 | 0 | 0 | 19 |
| SRR13616023 | 124 | 2781 | 166 | 166 |51 | 30 | 75 | 5894 |
| SRR13616024 | 186 | 60 | 350 | 12437 |58 | 13661 | 68 | 318 |
| SRR13616025 | 9 | 172 | 6 | 1 |0 | 0 | 1 | 269 |
| SRR13616026 | 0 | 1 | 0 | 0 |0 | 0 | 0 | 0 |
| SRR13616027 | 0 | 0 | 0 | 0 |0 | 0 | 0 | 0 |
| SRR13616028 | 804 | 55153 | 1250 | 550 |221 | 185 | 285 | 51570 |
| SRR13616029 | 0 | 2 | 0 | 1 |0 | 1 | 0 | 1 |
