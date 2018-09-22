# FLASHit

FLASH (Finding Low Abundance Sequences by Hybridization) is a method
using the targeted nuclease Cas9 to enrich for an arbitrary set of sequences
in a complex DNA library. FLASH-NGS achieves up to 5 orders of magnitude of
enrichment, sub-attomolar detection of genes of interest,
and minimal background from non-FLASH-derived reads.

FLASHit is open-source software for designing FLASH experiments.
It allows users to design guide RNA libraries for any set of target genes.
The guides are guaranteed to have good structure, and to avoid cutting a set of off-target genes,
such as human DNA in the case of a library designed to target resistant genes in pathogenic bacteria.

The problem of constructing a library efficiently, using the fewest guides
necessary to get the desired coverage of all target genes, is solved
as an mixed integer program optimization. For details, see the [formal description](docs/Guide_design.pdf).


The inputs consist of:

* fasta files of genes to be targeted
* fasta files of genes to be avoided

The output is:

* a list of RNA guides

To reproduce the library construction from the paper, after installing the
prerequisites below, run

`make amr_library`.

This will produce 2 files in `generated_files/untracked`:

* `library.txt` -- all optimized guides for genes used for the paper.
* `amr_library.txt` -- this is the set of guides used in the paper.

## Prerequisites

1) Install the [Anaconda](https://www.anaconda.com/) package manager for python.

2) Clone the FLASH repository from GitHub

	git clone https://github.com/chanzuckerberg/flash.git
	cd flash

3) Install the dependencies with conda.

	conda env create -f environment.yml

	source activate flash

3) License the Gurobi optimizer.

Get a license from [Gurobi](https://user.gurobi.com/download/licenses/free-academic). Academic licenses are free.

	grbgetkey KEYHERE

4) Install [GO](https://golang.org/doc/install).

If you wish to avoid the human genome as offtarget as in the paper, then

5) Download [hg38.fa.gz](http://hgdownload.cse.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz)
into `inputs/additional/offtargets/`.

	curl http://hgdownload.cse.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz > inputs/additional/offtargets/hg38.fa.gz

## Build Stages

### Build gene files

This step creates a standardized fasta file for each gene, with metadata
tags embedded in the header. These files are deposited in `generated_files/genes`.
This is automatic for genes coming from CARD and Resfinder;
for information about how to format your own
genes appropriately consult the Gene Formatting section below.

```
make build_gene_files
```

### Compile offtargets

This step extracts and collates all CRISPR targets from the offtarget files.
For the human genome, this takes about 2 minutes and produces a ~4 GB txt file.

```
make generate_offtargets
```

### Build gene indices

This step finds all the CRISPR targets in the input genes, filtered for structure and offtargets.

```
make build_indices
```

### Run optimizer

This step finds the optimal library of guides for targeting the input genes.

`make optimizer'

To search for guides extending a a given library, include that library as an additional input.

`python optimizer.py --extend [library.txt] --output [extended_library.txt]`

### Extract guides for a certain set of genes

`python extract_guides.py [library.txt] [gene_list.txt]`

## Analysis

To inspect a given gene--its list of potential
CRISPR cut sites, its SNPs, and the cuts made by a given library--run the
`display_genes.py` script:

`echo 'nalC__NC_002516__ARO_3000818' | python display_genes.py`

## AMR Example

FLASHit has been used to design guides to target antimicrobial resistance (AMR)
genes. Those genes were gathered from:

1) The Comprehensive Antibiotic Resistance Database ([CARD](https://card.mcmaster.ca/)) from McMaster University.
Those genes are in `inputs/card`.
2) [Resfinder](https://cge.cbs.dtu.dk/services/ResFinder/) from the Center for Genomic Epidemiology.
Those genes are in `inputs/resfinder`.
3) Additional resistance genes gathered from the literature.
Those genes are in `inputs/additional`.

## SNPs

The guides may be designed to capture certain SNPs (or regions) present in
each gene. Guides that would hybridize to those variable region are forbidden,
and we ensure that each variable region is contained in a fragment.

For some SNPs near the end of genes, additional genomic context is required so
that a target on each side of the SNP can be found.
That can be added by hand by pre/post-pending sequence to the gene (and adjusting
the locations for the SNPs accordingly). We organize this through
the use of padding: a simple `json` file containing the additional
sequence for those genes requiring it, `padding.json`.

## Git

For the paper, we keep many of the intermediate files in github for easy versioning.
All files in 'generated_files/under_version_control' are automatically committed
when generated. To review what has changed, run `git diff HEAD`.

If you would like to work with a different collection of genes,
SNPs, or offtargets, we recommend working in a new git branch. (To create a new branch,
run `git checkout -b BRANCHNAME`).

## Gene formatting

The header of each gene can contain arbitrary text separated by pipes (|).
It must also include key-value pairs specifying a unique id for the gene
(`flash_key`) and, if SNPs are to be targeted, the location of the SNPs
(`flash_mutation_ranges`).

`>NCTC_8325|ARO:3003312|Staphyloccocus_aureus_NCTC_8325_parC|flash_key:parC__NCTC_8325__fluoroquinolone_additional|flash_resistance:fluoroquinolone|flash_mutation_ranges:nt240+12`

Genes in the FLASH pipeline are canonically named with unique keys
that look like this

	GES-11__FJ854362__ARO_3002340
	ermD__M29832__macrolide
	fdg2__NC_000962.3__delamanid_additional

Each such key consists of 3 fields separated by double underscore.

The first field is a gene name, in the above example GES-11 or ermD or fdg2.

The second field is a LOCUS or accession number of some sort.

The third field is either

   -- an ARO number for CARD genes, or

   -- a Resfinder file name (usually an antibiotic name) for Resfinder genes, or

   -- a tag usually consisting of an antibiotic name and the word "additional",
	  for genes coming from the inputs/additional/*.fasta files

 These keys are inferred for genes from CARD and Resfinder, or user-supplied
 for genes from inputs/additional.   The rules are strictly enforced, so if
 you are adding manually a gene under inputs/additional, and you neglect to
 specify a valid unique key or resistance tag, your gene will be rejected.

 Here is an example.  Suppose you want to add the gene

 >NC_000962.3:490783-491793_fdg1 Mycobacterium tuberculosis H37Rv, complete genome
GTGGCTGAACTGAAGCTAGGTTACAAAGCATCGGCCGAACAATTCGCACCGCGCGAGCTCGTCGAACTAG
CCGTCGCCGCCGAAGCCCACGGCATGGACAGCGCGACCGTCAGCGACCATTTTCAGCCTTGGCGCCACCA
...

and you have the above in file inputs/additional/myantibiotic.fasta.  It will
be rejected unless you extend the header with two more tags, like so

>NC_000962.3:490783-491793_fdg1 Mycobacterium tuberculosis H37Rv, complete genome|flash_key:fdg1__NC_000962.3__myantibiotic_additional|flash_resistance:myantibiotic
GTGGCTGAACTGAAGCTAGGTTACAAAGCATCGGCCGAACAATTCGCACCGCGCGAGCTCGTCGAACTAG
CCGTCGCCGCCGAAGCCCACGGCATGGACAGCGCGACCGTCAGCGACCATTTTCAGCCTTGGCGCCACCA
...

"myantibiotic" should be replaced with the specific antibiotic that
this gene confers resistance to, and should match the file name as
well as the flash_resistance key added to the header above.

Note the addition of the flash_key with value

	fdg1__NC_000962.3__myantibiotic_additional

where again myantibiotic should be replaced with the name of the specific
antibiotic for this gene.
