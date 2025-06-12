# GOMA vAlpha 1.0 - architecture

The GOMA pipeline is built from multiple software chained together to create a full analysis of the input data.

<div style="display: flex; flex-direction: column; align-items: center;">
    <img src="/assets/Workflow_schematic_alpha.PNG" alt="Create history in Galaxy" style="margin-bottom: 5px; width: 640px;">
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

### Optional identification of transmission links
To investigate if the samples cluster together resembling a possible outbreak situation, the option "identify_transmission_links" can be chosen upon starting the pipeline. This branch of the workflow will create a SNP alignment and try to find clusters based on SNP distances between the samples. This happends regardless of whether there is an actual epidemiological link between the samples. This has to be kept in mind when interpreting the results. 

### Variant Filtering III
The variants are filtered differently for the identification of transmission clusters. **TB Variant filter (Galaxy Version 0.4.0)** is set to filter out the following regions: regions of low mappability, variants in the regions of the PE/PPE genes and antibiotic resistant genes according to TB-Profiler. The antibiotic resistance genes need to be filtered out because cluster analyses is based on genetic relatedness of the samples. A comparison of genetic differences without filtering out antibiotic resistance genes could confound transmission cluster analysis in instances where strains are genetically very closely related. We filter out any variants with an allele frequency below 10% and replace alternative bases of variants with a frequency between 10% and 90% with an 'N' to resemble ambiguity. This step is done with **Text reformatting with awk (Galaxy Version 9.3)**. INDELs can cause misalignments in their surrounding areas, leading to incorrect variant calls. We 
therefore let the option “Mask around INDELs” be at the default value of five, meaning that variants with a distance of 5 bases or less to INDELs are not considered. We again retained only variants with a minimum read depth of 7.

### SNP alignment
To create the SNP alignment (alignment of only the variable single base positions of all samples), **bcftools consensus (Galaxy Version 1.15.1)** creates the consensus genomes (recreated genome of each sample with the variants inserted in the reference genome). Changing variants with an allele frequency between 10% and 90% to 'N' (as done in the previous step), prevents the insertion of the ancestral base in the consensus genome and gives the opportunity to adress these positions downstream. The resulting genomes are concatenated into a multifasta alignment using **Concatenate datasets tail-to-head (cat) (Galaxy Version 9.3)**. The tool Find **SNP sites (Galaxy Version 2.5.1)** removes all invariant positions from the multifasta file, creating the SNP alignment in FASTA format.

### SNP distance matrix
To calculate the SNP distances between the samples, **SNP distance matrix (Galaxy Version 0.8.2)** computes pairwise distances in SNPs between all sequences in the multifasta SNP alignment.

### Cluster analysis
To find potential transmission clusters, distance matrix-based hierarchical clustering is performed using **Scipy (Galaxy Version 1.1)**. The UPGMA method is used for clustering, and the distance threshold for clusters is once set at 6 and once at 13, to report clusters with 5 or less SNPs and 12 or less SNPs distance.

