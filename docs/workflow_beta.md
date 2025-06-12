# GOMA vBeta 0.1 - in development - architecture

The GOMA pipeline is built from multiple software chained together to create a full analysis of the input data.

<div style="display: flex; flex-direction: column; align-items: center;">
    <img src="/assets/Workflow_schematic_beta.PNG" alt="Create history in Galaxy" style="margin-bottom: 5px; width: 640px;">
    <p style="text-align: center; margin: 0;">Flowchart showing the key pipeline blocks.</p>
</div>

## Key pipeline blocks
### Input files
The GOMA pipeline is designed to process raw Illumina FASTQ files. Both paired-end (PE) and single-end (SE) files can be processed. A mixed batch of PE and SE data is also possible. The sequencing data needs to be prepared in a certain way (see "[Running the GOMA pipeline](index.md/#data-preparation)")  which can be done in Galaxy as well. The reference genome and the annotation files are uploaded on Zenodo and [publicly available](https://zenodo.org/records/14536411). 

### Raw read processing
To process the raw FASTQ files, **fastp (Galaxy Version 0.23.3)** with default settings is used. Overall read qualities are assessed, residing adapters are trimmed and quality trimming is performed. For a read to be removed, more than 40% of bases of a read have to have a phred score of 15 or lower. If more than 5 bases 
of a read are labelled as “N” (unidentified by sequencing), the read is also removed. An overview of quality metrics of all FASTQ files both PE and SE is provided by **MultiQC (Galaxy version 1.24.1)**.

### Filtering contamination
The GOMA pipeline taxonomically classifies sequencing reads before mapping them to the reference genome. **Kraken2 (Galaxy Version 2.1.3)** is used to classify reads belonging to the *Mycobacterium tuberculosis* complex (MTBC) using a *Mycobacterium* specific database (Mycobacterium v1 in Galaxy). **Krakentools: Extract kraken reads by ID (Galaxy Version 1.2)** filters the FASTQ file only keeping reads classified as being MTBC, according to NCBI taxonomy browser's collection of MTBC taxonomic ID's (Oct 2024).

### Mapping and variant calling
**Snippy (Galaxy Version 4.6.0)** with default parameters performs mapping and variant calling in one step. BWA --MEM is used as the mapping software and FreeBayes does the variant calling. The inferred ancestral MTBC genome, based on the chromosome of H37Rv (Comas et al., 2010; NC_000962.3), is used as the reference. This step produces three files: The variants in VCF format, a summary table of all called variants and the alignments in BAM format, excluding multi-mapped and ummapped reads.

### All VCF files & All BAM files
In the case of the pipeline being run on both PE and SE FASTQ files, all SE & PE VCF files and all SE & PE BAM files get now merged into a VCF-only and BAM-only collection. After mapping and variant calling, the operations perfomed by the pipeline can be done with the same parameters irrespectible of the underlying FASTQ files being SE or PE. This step is done with the Galaxy tool **Merge collections" (Galaxy Version 1.0.0)**. Failed datasets are filtered out ensuring a smooth ongoing of the pipeline's flow with the tool **Filter failed datasets" (Galaxy Version 1.0.0)**.

### Variant Filtering I
All the variant filtering is done with **TB Variant Filter (Galaxy Version 0.4.0)**. The minimum allele frequency to call a variant is 10%, aloowing the capture of variants in the case of a heterogeneous bacteria population, e.g. a mixed infection. The minimum read depth was is set at 7. Variants in the following regions are filtered out: regions of low mappability as identified in Marin et al (Marin et al.,2022), and variants in the regions of the PE/PPE genes (Fishbein et al., 2015).

### Variant annotation
**SnpEff (Galaxy Version 4.3t)** annotates the variants wit a custom database built using **SnpEff build (Galaxy Version 4.3t)** and the provided GFF file and the reference genome. SnpEff also uses a custom interval annotation file in BED format to annotate variants linked to essential genes (DeJesus et al., 2017), drug resistance (Walker et al., 2015) and functional categories from MycoBrowser (Kapopoulou et al., 2011). SnpEff runs on default parameters except not showing upstream and downstream changes. The output is the annotated VCF and a HTML summary report.

### TB-Profiler
To create the drug-resistance report, **TB-Profiler Profile (Galaxy Version 6.2.1+galaxy1)** is run on the BAM file from snippy. The mapping software of TB-Profiler is set to BWA and GATKv4 is chosen as TB-Profiler's variant caller. The custom database of TB-Profiler used in by GOMA is based on the second WHO catalogue for DR-associated mutations in the MTBC. TB-Profiler creates drug-resistance profiles, lineage reports per sample and a VCF file containing the resistance variants.

### Lineage and variant detection, TB Variant report
**TB Variant report (Galaxy Version 1.0.1)** creates final reports on lineage, variants and drug-resistances by parsing the annotated VCF and the TB-Profiler drug-resistance report. The variant report consists of information on lineage, sub-lineage, family, RD (regions of difference, important genetic markers for different MTBC clades) and a table of all annotated variants. The drug-resistance report additionally creates a table with known drugs used to treat TB with information on whether the sample is predicted to be resistant against these antibiotics and the supporting mutations of these predictions. **JBrowse (Galaxy Version 1.16.11)** visualizes the found variants using three track groups set to sequnce reads (Input: BAM file), Variants (Input: Filtered VCF file) and annotated reference (Input: GFF file).

### Variant filtering II and variant calling validation
Statistics on the created VCF files are generated with **bcftools stats (Galaxy Version 1.15.1)**. For this step, we filtered the VCF files for variants with an allele frequency of at least 90%. This step can be used to validate the variant calling process by comparing the number of variants of bcftools stats with those in a known reference dataset.

!!! note "Changes from vALpha to vBeta" 

    ### Optional identification of transmission links and phylogenetic analysis
    To investigate if the samples cluster together resembling a possible outbreak situation and or to investigate the phylogenetic relationships between the samples, the option "transmission_and_phylogenetic_analysis can be chosen upon starting the pipeline. This branch of the workflow will create an alignment of variable positions and try to find clusters based on SNP distances between the samples. This happends regardless of whether there is an actual epidemiological link between the samples. This has to be kept in mind when interpreting the results. The alignment also serves as the base for building the phylogenetic tree, for which *M. canettii* is set as the outgroup.

    ### TBConsensusAligner
    To create the alignment necessary for transmission cluster analysis and the phylogenetic tree, a custom script called **TBConsensusAligner** is used. This script creates consensus genomes and subsequently the variable alignment. The consensus genomes are created based on the VCF files, the reference genome and the information on depth of each position per sample. This information is obtained with **samtools depth (Galaxy Version 1.15.1+galaxy2)**. BED files can be used to mask certain regions. BED files to mask regions of low mappability, PPE/PE genes and TB-Profilers antibiotic resistance genes are provided on zenodo. The script produces the consensus genomes while taking into account SNPs and MNPs, large and small deletions, small insertions and sites of low quality. For variants with an allele frequency of >=90%, the VCF's variant is considered, variants with an allele frequency between 10% and 90% the ambiguity base 'N' is set and for variants with an allele frequency of below 10% the ancestral state of the reference genome is set.

    The alignment of polymorphic positions is produced based on the created consensus genomes. All consensus genomes are being scanned per genomic position. If a position is polymorphic, i.e. has more than one base across all genomes that is not 'N' or a gap, it is written to the output. This creates an alignment of all polymorphic positions across all genomes. For each such position, the corresponding base from the VCF of *M. canettii* is extracted, creating the outgroup. By default the allowed proportion of undefined states (N's or gaps) for a polymorphic position to be kept in the alignment is set at 90%, but this can be changed by the user.

    ### SNP distance matrix
    To calculate the SNP distances between the samples, SNP distance matrix (Galaxy Version 0.8.2) computes distances in SNPs between all sequences in the multifasta file.

    ### Cluster analysis
    To find potential transmission clusters, distance matrix-based hierarchical clustering is performed using Scipy (Galaxy Version 1.1). The UPGMA method is used for clustering, and the distance threshold for clusters is once set at 6 and once at 13, to report clusters with 5 or less SNPs and 12 or less SNPs distance.

    ### Phylogenetic analysis
    The alignemt of polymorphic positions is passed to **IQ-Tree (galaxy Version 2.3.6)** with *M. canettii* set as the outgroup. IQ-Tree runs with default settings except we keep identical sequences and use GTR as a custom model. Ultrafast bootstrap parameter is run with 1000 replicates. To visualize the tree, the makimal likelihood tree (.nhx format) can be uploaded to [iTOL](https://itol.embl.de/). 


