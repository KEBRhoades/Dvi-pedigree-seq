# Persimmon Pedigree Sequencing Project: Methods

## Sequencing Info

From Meghan at Neochromosome 4/22:

> DNA extraction: DNA was extracted from lyophilized leaf tissue punches
> with the MagMAX Plant DNA Kit (Life Technologies cat# A32549), using
> the manufacturer’s recommendations for processing samples in 96-well
> plate format on the Kingfisher (ThermoFisher) instrument. The
> following modification was made to the manufacturers protocol: PVP was
> added to Lysis Buffer A to a final concentration of 1%.

> NGS Library prep for Whole Genome Sequencing (WGS): Sample DNA was
> normalized to ~1ng/µl per sample in a 96-well plate prior to Library
> prep. The Illumina DNA Prep (M) Tagmentation (Illumina 20060059)
> workflow was used for preparation of WGS libraries, with
> modifications. Library prep reactions were miniaturized to 10µl, 6ng
> of DNA per sample was used as input, and the resulting libraries were
> eluted in 10µl Qiagen Buffer EB. Libraries were quantified using the
> High-Sensitivity dsDNA Qubit Assay, and their fragment sizes estimated
> using the High-Sensitivity D5000 Tapestation Assay.

> Library sequencing: Sample libraries were sequenced to a depth of 20x
> on the Illumina NovaSeqX Plus instrument, using PE 150bp format reads.

## Environment info

This part of the work is being done on AWS servers running Ubuntu. For
the individual software tools I’m creating conda environments.

## Read Quality Trimming

FastQC of reads before trimming looked pretty good, honestly, except for
low quality at the first and last 18 bases of the reads and Nextera
adapters. Had some trouble with my usual beloved Trimmomatic so on the
advice of the Neochromosome folks I switched over to
[Trim-Galore](https://github.com/FelixKrueger/TrimGalore), which is a
wrapper for Cutadapt and FastQC.

    trim_galore -q 30 --nextera --fastqc --cores 8 --paired --clip_R1 18 --clip_R2 18 --basename earlygolden_t18 [rawreads1.fq.gz] [rawreads2.fq.gz]

A couple early runs I did not trim the first and last 18 bases so all
the ones where I did I noted t18 in the filename. At some point I do
want to compare SNPS/coverage/etc between the Early Golden where I just
trimmed adapters and the Early Golden where I cut first/last 18 bases.

## Alignment to Early Golden genome

**I FORGOT TO ADD READ GROUPS GAHHH**

Aligning everything to `Dv_EG.main-genome.fasta` for the moment.

Using the classic: [bwa](https://bio-bwa.sourceforge.net/bwa.shtml)

    bwa mem -t 32 -R '@RG\tID:earlygolden' D.virginiana_genome/Dv_EG.main-genome.fasta trimmed_reads/earlygolden_t18_val_1.fq.gz trimmed_reads/earlygolden_t18_val_2.fq.gz > alignments/earlygolden_t18.Dv-EG.main.sam

The above is the way I should have done it, with read group line
specified (this IDs the alignments and then later the vcf file so that I
can combine vcf files of multiple samples together). Instead I’m adding
read group lines to the .sam/.bam files afterwards with samtools like a
DOOF.

\*also if I end up having to redo bwa mem maybe add an -a tag in as
well, to add secondary alignments into the mix?

#### Adding read groups after alignment like a doof

`samtools addreplacerg -@ 4 -r "@RG\tID:G04_1\tPL:ILLUMINA\tLB:britainsblue\tSM:britainsblue" -o britainsblue_t18_md_rg.Dv-EG.main.bam britainsblue_t18_md.Dv-EG.main.bam`

**I don’t think doing this after marking duplicates screws anything up
but I will triple-check that.**

Update 5/2: So long as there was only one RG per file (which there was),
it’s okay that I did MarkDuplicates before adding read groups to some of
these. [Biostar link](https://www.biostars.org/p/9487148/)

## Mark duplicates

I am doing this with samtools because it is less annoying than GATK
(although GATK would have alerted me that I forgot to do read group
labeling so…who can say)

    samtools sort -n -@ 8 f100female_t18.Dv-EG.main.sam | samtools fixmate -cm -O bam -@ 8 - f100female_t18_fixmate.Dv-EG.main.bam

    samtools sort -@ 8 f100female_t18_fixmate.Dv-EG.main.bam | samtools markdup -l 133 -s -f f100female_mdstats.txt -@ 8 - f100female_t18_md.Dv-EG.main.bam

## Identify variants with freebayes

Okay so I did my several initial freebayes runs with the defauls
(`freebayes -f ref.fa aln.bam > var.vcf`) but now that I’m going to have
to redo them because of the read groups thing I think I’m going to take
advantage of more of the filtering.

    freebayes -f ref.fa --gvcf --standard-filters aln.bam > var.vcf

Standard filters are:

-   min mapping quality (-m) = 30
-   min base quality (-q) = 20
