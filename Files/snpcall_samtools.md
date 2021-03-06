
The commands provided below refer to a previous version of SAMtools and may not be functional with the most recent version.

A commonly used program SAMtools, which outputs a file called VCF (Variant Call Format). 
More information can be found [here](http://www.ncbi.nlm.nih.gov/pubmed/21653522) and [here](http://vcftools.sourceforge.net/specs.html).

VCF is becoming the standard format for storing variable sites from sequencing data. 
It works with SNPs, indels and structural variations.
VCF files have a header and a data section. The header contains a line starting with '#' with the name of each field, plus several lines starting with '##' containing additional information. 
The data section is TAB delimited and each line has at the least the first 8 of these fields:

name | value
---- | -----
CHROM | Chromosome name
POS | Position (1-based)
ID | Identifier
REF | Reference (base)
ALT | Alternative (base)
QUAL | Probability of all samples being homozygous reference (and thus not variable)
FILTER | List of filter that this variant failed to pass
INFO | List of misc information
FORMAT | Format of the individual genotypes
... | Individual genotypes

By default, SAMtools produces a binary version of VCF files, called BCF. 
To get and view a VCF version, you need to use bfctools, a utility within the SAMtools package.

We will use the human dataset of 33 samples as an illustration for calling SNPs using SAMtools.
We first need to index the BAM files and the reference sequence to allow for random access, and make a list with the newly indexed BAM files.
You do not need to run this since all files are available in the `input/` folder.
```
for i in input/human/smallNA*.bam; do ./samtools-0.1.19/samtools index $i;done
ls input/human/smallNA*.bam > input/human/bams.list
./samtools-0.1.19/samtools faidx input/human/hg19_chr1.fa.gz
```

Let us first see all options for **mpileup**:
```
./samtools-0.1.19/samtools mpileup

Usage: samtools mpileup [options] in1.bam [in2.bam [...]]

Input options:

       -6           assume the quality is in the Illumina-1.3+ encoding
       -A           count anomalous read pairs
       -B           disable BAQ computation
       -b FILE      list of input BAM filenames, one per line [null]
       -C INT       parameter for adjusting mapQ; 0 to disable [0]
       -d INT       max per-BAM depth to avoid excessive memory usage [250]
       -E           recalculate extended BAQ on the fly thus ignoring existing BQs
       -f FILE      faidx indexed reference sequence file [null]
       -G FILE      exclude read groups listed in FILE [null]
       -l FILE      list of positions (chr pos) or regions (BED) [null]
       -M INT       cap mapping quality at INT [60]
       -r STR       region in which pileup is generated [null]
       -R           ignore RG tags
       -q INT       skip alignments with mapQ smaller than INT [0]
       -Q INT       skip bases with baseQ/BAQ smaller than INT [13]
       --rf INT     required flags: skip reads with mask bits unset []
       --ff INT     filter flags: skip reads with mask bits set []

Output options:

       -D           output per-sample DP in BCF (require -g/-u)
       -g           generate BCF output (genotype likelihoods)
       -O           output base positions on reads (disabled by -g/-u)
       -s           output mapping quality (disabled by -g/-u)
       -S           output per-sample strand bias P-value in BCF (require -g/-u)
       -u           generate uncompress BCF output

SNP/INDEL genotype likelihoods options (effective with `-g` or `-u`):

       -e INT       Phred-scaled gap extension seq error probability [20]
       -F FLOAT     minimum fraction of gapped reads for candidates [0.002]
       -h INT       coefficient for homopolymer errors [100]
       -I           do not perform indel calling
       -L INT       max per-sample depth for INDEL calling [250]
       -m INT       minimum gapped reads for indel candidates [1]
       -o INT       Phred-scaled gap open sequencing error probability [40]
       -p           apply -m and -F per-sample to increase sensitivity
       -P STR       comma separated list of platforms for indels [all]
```

and **bfctools**
```
./samtools-0.1.19/bcftools/bcftools view

Usage: bcftools view [options] <in.bcf> [reg]

Input/output options:

       -A        keep all possible alternate alleles at variant sites
       -b        output BCF instead of VCF
       -D FILE   sequence dictionary for VCF->BCF conversion [null]
       -F        PL generated by r921 or before (which generate old ordering)
       -G        suppress all individual genotype information
       -l FILE   list of sites (chr pos) or regions (BED) to output [all sites]
       -L        calculate LD for adjacent sites
       -N        skip sites where REF is not A/C/G/T
       -Q        output the QCALL likelihood format
       -s FILE   list of samples to use [all samples]
       -S        input is VCF
       -u        uncompressed BCF output (force -b)

Consensus/variant calling options:

       -c        SNP calling (force -e)
       -d FLOAT  skip loci where less than FLOAT fraction of samples covered [0]
       -e        likelihood based analyses
       -g        call genotypes at variant sites (force -c)
       -i FLOAT  indel-to-substitution ratio [-1]
       -I        skip indels
       -m FLOAT  alternative model for multiallelic and rare-variant calling, include if P(chi^2)>=FLOAT
       -p FLOAT  variant if P(ref|D)<FLOAT [0.5]
       -P STR    type of prior: full, cond2, flat [full]
       -t FLOAT  scaled substitution mutation rate [0.001]
       -T STR    constrained calling; STR can be: pair, trioauto, trioxd and trioxs (see manual) [null]
       -v        output potential variant sites only (force -c)

Contrast calling and association test options:

       -1 INT    number of group-1 samples [0]
       -C FLOAT  posterior constrast for LRT<FLOAT and P(ref|D)<0.5 [1]
       -U INT    number of permutations for association testing (effective with -1) [0]
       -X FLOAT  only perform permutations for P(chi^2)<FLOAT [0.01]
```

A typical command line for SNP calling can be (recalling what already discussed on reads filtering):
```
./samtools-0.1.19/samtools mpileup -f input/human/hg19_chr1.fa.gz -b input/human/bams.list -C 50 -q 20 -Q 20 -g -r 1: -I | ./samtools-0.1.19/bcftools/bcftools view -c -g -v -d 0.5 -I - > output/human.vcf
```
You can now have a look at the output file. How many variable sites does it output?
```
less -S output/human.vcf
```

We can change the parameters for SNP calling and check the results. For instance we can use a more stringent cutoff:
```
./samtools-0.1.19/samtools mpileup -f input/human/hg19_chr1.fa.gz -b input/human/bams.list -C 50 -q 20 -Q 20 -g -r 1: -I | ./samtools-0.1.19/bcftools/bcftools view -c -g -v -d 0.5 -I -p 0.1 - > output/human.v2.vcf
```
How many variable sites does it output now?
Look at the sites that are now not predicted to be variable anymore using this new cutoff. 
Why are they SNPs anymore?

As an optional exercise, you can try different thresholds and print (or tabulate) the corresponding number of predicted number of SNPs.

-----------

Here we want to compare SNP calling using ANGSD and SAMtools.
Results from these programs are not entirely comparable in a strict way, but here we just want to show what happens if you simply use default values in SAMtools to call SNPs.

We will use `R`, and we need the `VennDiagram` package installed. 
A simple script is provided to plot a Venn diagram of the overlapping SNPs, as well as text files with the indexes of such sites.

We use the butterfly dataset, so let us quickly call SNPs using ANGSD:
```
./angsd/angsd -b input/lyca/bams.list -ref input/lyca/referenceseq.fasta -GL 1 -doMajorMinor 1 -doMaf 2 -SNP_pval 0.001 -sites input/lyca/sites.bed -out output/lyca.snp1 &> /dev/null
./angsd/angsd -b input/lyca/bams.list -ref input/lyca/referenceseq.fasta -GL 1 -doMajorMinor 1 -doMaf 2 -SNP_pval 0.0001 -sites input/lyca/sites.bed -out output/lyca.snp2 &> /dev/null
```
and SAMtools, without worrying too much about tuning options:
```
./samtools-0.1.19/samtools mpileup -f input/lyca/referenceseq.fasta -b input/lyca/bams.list -l input/lyca/sites.sam.bed -g -I | ./samtools-0.1.19/bcftools/bcftools view -c -g -v -d -I - > output/lyca.samtools.vcf 2> /dev/null
```
Note that ANGSD is faster than SAMtools (although the latter performs more analyses in this run).

Let us compare these results using `R`.
```
Rscript scripts/plotSNPcall.R output/lyca.samtools.vcf output/lyca.snp1.mafs.gz output/lyca.snp2.mafs.gz > output/lyca.snpcall.compare.txt
open output/lyca.snpcall.pdf
```

Look at the sites that SAMtools calls variable but not ANGSD, and viceversa:
```
less -S output/lyca.snpcall.compare.txt
```










