---
output:
  html_document: default
  pdf_document: default
---
Genome-wide DNA binding in HEPG2
================
Friedman, A., Giarratana, D., Martinez, P., Steiner, H., Rinn, J.
4/25/2022

# Goal:

Here we aim to download all available DNA binding protein (DBP) profiles
in a single cell state (measured by ChIP-seq) . This will allow us to
investigate the binding properties of hundreds of DBPs in the same
cellular context or background. We aim to address several questions: (i)
What are the number of peaks and genome coverage for each DBP? (ii) What
are the binding preferences for promoters, gene-bodies and intergenic
genomic regions? (iii) What are the similarities and differences across
DBPs based on their genome-wide binding profiles? (iv) What properties
or preferences do promoters have for binding events. (iv) Are there
reservoir promoters in HepG2 as defined in k562 previously? (v) How does
binding to a promoter affect the transcriptional output of that
promoter?

To address these questions we have curated a set of X,000 ChIPs-eq data
sets comprised of 486 DBPs in HEPG2 cells from the ENCODE consortrium.
We required duplicate ChIP-seq experiments for a given DBP and other
criterion that can be found here :

<https://www.encodeproject.org/report/?type=Experiment&status=released&assay_slims=DNA+binding&biosample_ontology.term_name=HepG2&assay_title=TF+ChIP-seq&biosample_ontology.classification=cell+line&files.read_length=100&files.read_length=76&files.read_length=75&files.read_length=36&assay_title=Control+ChIP-seq&assay_title=Histone+ChIP-seq&files.run_type=single-ended>

## These samples were selected on the following criteria:

1.  “chromatin” interaction data, then DNA binding data, cell line
    HEPG2, “TF-Chip-seq”.
2.  We further selected “TF Chip-seq”, “Control chip-seq” and “Histone
    Chip-seq”.
3.  We selected several read lengths to get the most DNA binding
    proteins (DBPs)
4.  Read lengths: 100, 76, 75, 36
5.  ONLY SINGLE END READS (this eliminates 54 samples)

### Experimental data was downloading by (ENCODE report.tsv):

<https://www.encodeproject.org/report.tsv?type=Experiment&status=released&assay_slims=DNA+binding&biosample_ontology.term_name=HepG2&assay_title=TF+ChIP-seq&biosample_ontology.classification=cell+line&files.read_length=100&files.read_length=76&files.read_length=75&files.read_length=36&assay_title=Control+ChIP-seq&assay_title=Histone+ChIP-seq&files.run_type=single-ended>

### The FASTQ files were downloaded with:

“<https://www.encodeproject.org/metadata/?status=released&assay_slims=DNA+binding&biosample_ontology.term_name=HepG2&assay_title=TF+ChIP-seq&biosample_ontology.classification=cell+line&files.read_length=100&files.read_length=76&files.read_length=75&files.read_length=36&assay_title=Control+ChIP-seq&assay_title=Histone+ChIP-seq&files.run_type=single-ended&type=Experiment>”

MD5sums were checked with all passing (see encode\_file\_info function
to reterive MD5Sum values that are not available from the encode portal
(/util)

### Processing data:

We processed all the read alignments and peak calling using the NF\_CORE
ChIP-seq pipeline: (nfcore/chipseq v1.2.1)

## Next we created consensus peaks that overlap in both replicates

Our strategy was to take peaks in each replicate and find all
overlapping peak windows. We then took the union length of the
overlapping range in each peak window.

``` r
# create_consensus_peaks requires an annotation .GTF file - loading in Gencode v32 annotations.
gencode_gr <- rtracklayer::import("/scratch/Shares/rinnclass/CLASS_2022/data/genomes/gencode.v32.annotation.gtf")

# Creating consensus peaks function to create a .bed file of overlapping peaks in each replicate.
# /util/intersect_functions.R

# TODO run this only on final knit
# create_consensus_peaks <- create_consensus_peaks(broadpeakfilepath = "/scratch/Shares/rinnclass/CLASS_2022/data/test_work/all_peak_files")

# exporting consensus peaks .bed files

# TODO run this only on final knit
# for(i in 1:length(consensus_peaks)) {
#  rtracklayer::export(consensus_peaks[[i]],
#                     paste0("/scratch/Shares/rinnclass/CLASS_2022/JR/CLASS_2022/class_exeRcises/analysis/11_consensus_peaks/consensus_peaks/",
#                             names(consensus_peaks)[i],
#                             "_consensus_peaks.bed"))
# }
```

# loading in consensus peaks to prevent rerunning create\_consensus\_peaks function

``` r
# Loading in files via listing and rtracklayer import
consensus_fl <- list.files("/scratch/Shares/rinnclass/CLASS_2022/JR/CLASS_2022/class_exeRcises/analysis/11_consensus_peaks/consensus_peaks", full.names = T)

# importing (takes ~5min)
consensus_peaks <- lapply(consensus_fl, rtracklayer::import)

# cleaning up file names
names(consensus_peaks) <- gsub("/scratch/Shares/rinnclass/CLASS_2022/JR/CLASS_2022/class_exeRcises/analysis/11_consensus_peaks/consensus_peaks/|_consensus_peaks.bed","", consensus_fl)

# Filtering consensus peaks to those DBPs with at least 250 peaks
num_peaks_threshold <- 250
num_peaks <- sapply(consensus_peaks, length)
filtered_consensus_peaks <- consensus_peaks[num_peaks > num_peaks_threshold]

# Result: these were the DBPs that were filtered out.
filtered_dbps <- consensus_peaks[num_peaks < num_peaks_threshold]
names(filtered_dbps)
```

    ##  [1] "CEBPZ"    "GPBP1L1"  "H3K27me3" "HMGA1"    "IRF3"     "MLLT10"  
    ##  [7] "MYBL2"    "NCOA5"    "RNF219"   "RORA"     "ZBTB3"    "ZFP36"   
    ## [13] "ZFP62"    "ZMAT5"    "ZNF10"    "ZNF17"    "ZNF260"   "ZNF382"  
    ## [19] "ZNF48"    "ZNF484"   "ZNF577"   "ZNF597"   "ZNF7"

``` r
# We have this many remaining DBPs
length(filtered_consensus_peaks)
```

    ## [1] 460

# Now we will determine the peak number and genome coverage for each DBP.

``` r
# Let's start with loading in the number of peaks each DBP has -- using length.
num_peaks_df <- data.frame("dbp" = names(filtered_consensus_peaks),
                           "num_peaks" = sapply(filtered_consensus_peaks, length))

# total genomic coverage of peaks for each dbp
num_peaks_df$total_peak_length <- sapply(filtered_consensus_peaks, function(x) sum(width(x)))

# Plotting distribution of peak number per dbp
hist(num_peaks_df$total_peak_length)
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/peak%20number%20and%20coverage%20per%20DBP-1.png)<!-- -->

# Now we will create promoter annotations for lncRNA and mRNA and both.

We have created a funciton get\_promter\_regions that has up and
downstream parameters

``` r
# creating lncRNA and mRNA promoters
lncrna_mrna_promoters <- get_promoter_regions(gencode_gr, c("lncRNA", "protein_coding"), upstream = 2.5e3, downstream = 2.5e3)
names(lncrna_mrna_promoters) <- lncrna_mrna_promoters$gene_id
rtracklayer::export(lncrna_mrna_promoters, "analysis/results/lncrna_mrna_promoters.gtf")

# creating lncRNAs promoter
lncrna_promoters <- get_promoter_regions(gencode_gr, biotype = "lncRNA", upstream = 2.5e3, downstream = 2.5e3) 
names(lncrna_promoters) <- lncrna_promoters$gene_id
rtracklayer::export(lncrna_promoters, "analysis/results/lncRNA_promoters.gtf")

# creating mRNA promoters
mrna_promoters <- get_promoter_regions(gencode_gr, biotype = "protein_coding", upstream = 2.5e3, downstream = 2.5e3)
names(mrna_promoters) <- mrna_promoters$gene_id
rtracklayer::export(lncrna_promoters, "analysis/results/mRNA_promoters.gtf")

# creating all genebody annotation
lncrna_mrna_genebody <- gencode_gr[gencode_gr$type == "gene" & 
                                     gencode_gr$gene_type %in% c("lncRNA", "protein_coding")]
names(lncrna_mrna_genebody) <- lncrna_mrna_genebody$gene_id
rtracklayer::export(lncrna_mrna_genebody, "analysis/results/lncrna_mrna_genebody.gtf")

# creating lncRNA genebody annotation
lncrna_genebody <- gencode_gr[gencode_gr$type == "gene" & 
                                gencode_gr$gene_type %in% c("lncRNA")]
names(lncrna_genebody) <- lncrna_genebody$gene_id
rtracklayer::export(lncrna_mrna_genebody, "analysis/results/lncrna_genebody.gtf")

# creating mRNA genebody annotation
mrna_genebody <- gencode_gr[gencode_gr$type == "gene" & 
                              gencode_gr$gene_type %in% c("protein_coding")]
names(mrna_genebody) <-mrna_genebody$gene_id
rtracklayer::export(lncrna_mrna_genebody, "analysis/results/mrna_genebody.gtf")
```

# Determining the overlaps of chip peaks with promoters and genebodys

``` r
# creating index to subset lncRNA and mRNA annotations
lncrna_gene_ids <- lncrna_mrna_genebody$gene_id[lncrna_mrna_genebody$gene_type == "lncRNA"]
mrna_gene_ids <- lncrna_mrna_genebody$gene_id[lncrna_mrna_genebody$gene_type == "protein_coding"]

# using count peaks per feature returns number of annotation overlaps for a given DBP (takes ~5min)
promoter_peak_counts <- count_peaks_per_feature(lncrna_mrna_promoters, filtered_consensus_peaks, type = "counts")

# adding data to num_peaks_df
num_peaks_df$peaks_overlapping_promoters <- rowSums(promoter_peak_counts)
num_peaks_df$peaks_overlapping_lncrna_promoters <- rowSums(promoter_peak_counts[,lncrna_gene_ids])
num_peaks_df$peaks_overlapping_mrna_promoters <- rowSums(promoter_peak_counts[,mrna_gene_ids])

# gene body overlaps 
genebody_peak_counts <- count_peaks_per_feature(lncrna_mrna_genebody, 
                                                filtered_consensus_peaks, 
                                                type = "counts")


# adding data to num_peaks_df
num_peaks_df$peaks_overlapping_genebody <- rowSums(genebody_peak_counts)
num_peaks_df$peaks_overlapping_lncrna_genebody <- rowSums(genebody_peak_counts[,lncrna_gene_ids])
num_peaks_df$peaks_overlapping_mrna_genebody <- rowSums(genebody_peak_counts[,mrna_gene_ids])

write_csv(num_peaks_df, "analysis/results/num_peaks_df.csv")
```

# Plotting peak annotation features for DBPs

``` r
num_peaks_df <- read_csv("analysis/results/num_peaks_df.csv")
# Distriubtion of peak numbers of all 460 DBPs
ggplot(num_peaks_df, aes(x = num_peaks)) + 
  geom_histogram(bins = 70)
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/plotting%20peak%20annotation%20features-1.png)<!-- -->

``` r
# Plotting number of peaks versus total genome coverage
ggplot(num_peaks_df, aes(x = num_peaks, y = total_peak_length)) +
  geom_point() + 
  geom_smooth(method = "gam", se = TRUE, color = "black", lty = 2)+
         
  ylab("BP covered") +
  xlab("Number of peaks") +
  ggtitle("Peak count vs. total bases covered")
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/plotting%20peak%20annotation%20features-2.png)<!-- -->

``` r
ggsave("analysis/figures/peak_num_vs_coverage.pdf")


# Plotting number of peaks versus peaks overlapping promoters
ggplot(num_peaks_df,
       aes(x = num_peaks, y = peaks_overlapping_promoters)) +
  xlab("Peaks per DBP") +
  ylab("Number of peaks overlapping promoters") +
  ggtitle("Relationship Between Number of DBP Peaks and Promoter Overlaps")+
  geom_point() +
  geom_abline(slope = 1, linetype="dashed") +
  geom_smooth(method = "lm", se=FALSE, formula = 'y ~ x',
              color = "#a8404c") +
  stat_regline_equation(label.x = 35000, label.y = 18000) +
  ylim(0,60100) +
  xlim(0,60100)
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/plotting%20peak%20annotation%20features-3.png)<!-- -->

``` r
ggsave("analysis/figures/3_peak_num_vs_promoter_coverage.pdf")


# Plotting peak overlaps with genebody
ggplot(num_peaks_df,
       aes(x = num_peaks, y = peaks_overlapping_genebody)) +
  xlab("Peaks per DBP") +
  ylab("Number of peaks overlapping genes") +
  ggtitle("Relationship Between Number of DBP Peaks and Gene Body Overlaps")+
  geom_point() +
  geom_abline(slope = 1, linetype="dashed") +
  geom_smooth(method = "lm", se=F, formula = 'y ~ x',
              color = "#a8404c") +
  stat_regline_equation(label.x = 35000, label.y = 18000) +
  ylim(0,60100) +
  xlim(0,60100)
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/plotting%20peak%20annotation%20features-4.png)<!-- -->

``` r
ggsave("analysis/figures/4_peak_num_vs_gene_body_coverage.pdf")

# there is a large amount of data explained (almost all by genebodys)
# Let's see what percentage of the genome genebodys cover:

reduced_gene_bodies <- gencode_gr[gencode_gr$type == "gene"] %>%
  GenomicRanges::reduce() %>%
  width() %>%
  sum()

# percentage of gene bodies in genome
reduced_gene_bodies/3.2e9
```

    ## [1] 0.589159

# Counting the number of overlaps at each promoter

promoters are the cols and DBPs rows thus we can retrieve the number of
binding events at each promoter unlike the “counts parameter” that just
gives total number of overlaps

``` r
# Creating matrix of promoters(annotation feature) as cols and DBPs as rows (takes ~5min)
promoter_peak_occurence <- count_peaks_per_feature(lncrna_mrna_promoters, filtered_consensus_peaks, 
                                               type = "occurrence")

# test to make sure everything is in right order
stopifnot(all(colnames(promoter_peak_occurence) == lncrna_mrna_promoters$gene_id))

# Formatting final data.frame from peak occurence matrix
peak_occurence_df <- data.frame("gene_id" = colnames(promoter_peak_occurence),
                                "gene_name" = lncrna_mrna_promoters$gene_name,
                                "gene_type" = lncrna_mrna_promoters$gene_type,
                                "chr" = lncrna_mrna_promoters@seqnames,   
                                "1.5_kb_up_tss_start" = lncrna_mrna_promoters@ranges@start,
                                "strand" = lncrna_mrna_promoters@strand,
                                "number_of_dbp" = colSums(promoter_peak_occurence))
# exporting
write_csv(peak_occurence_df, "analysis/results/peak_occurence_dataframe.csv")
```

## ChIP peak calls distribute bimodally

``` r
ggplot(peak_occurence_df, aes(x = number_of_dbp)) +
geom_density(alpha = 0.2, color = "#424242", fill = "#424242") +
  
  theme_paperwhite() +
  xlab(expression("Number of DBPs")) +
  ylab(expression("Density")) +
  ggtitle("Promoter binding events",
          subtitle = "mRNA and lncRNA genes") +
  theme_paperwhite()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/bimodal%20promoter%20plot-1.png)<!-- -->

``` r
ggsave("analysis/figures/promoter_binding_events_lncrna_mrna.pdf")
```

There appears to be a host of promoters with a high number of protein
binding events (\~300).

# RNA-seq reveals high binders with unusually high expression

### Initialize samples for RNA-seq

``` r
# Create needed directories
system("cd /scratch/Shares/rinnclass/CLASS_2022/dagi0698/CLASS_2022/class_exeRcises/analysis/Genome-Wide_DNA_Binding_HEPG2")
system("mkdir fastq analysis results")

# Get sample list
system("wget -O samples.txt 'https://www.encodeproject.org/report.tsv?type=Experiment&status=released&assay_slims=Transcription&assay_slims=Transcription&replicates.library.biosample.donor.organism.scientific_name=Homo+sapiens&biosample_ontology.term_name=HepG2&biosample_ontology.classification=cell+line&assay_title=total+RNA-seq&files.read_length=50&limit=all&advancedQuery=date_released:[2009-01-01%20TO%202021-12-31]'")

# Get files
system("wget -O files.txt 'https://www.encodeproject.org/batch_download/?type=Experiment&status=released&assay_slims=Transcription&assay_slims=Transcription&replicates.library.biosample.donor.organism.scientific_name=Homo+sapiens&biosample_ontology.term_name=HepG2&biosample_ontology.classification=cell+line&assay_title=total+RNA-seq&files.read_length=50&limit=all&advancedQuery=date_released:[2009-01-01%20TO%202021-12-31]'")

# Copy files.txt to ./fastq/
system("cp files.txt fastq")

# Download data from ENCODE
system("cd fastq; xargs -L 1 curl -O -J -L < files.txt; cd ..")
```

### Prepare samples for RNA-seq

``` r
# Prepare sample list for RNA-seq
samples <- read.table("samples.txt",
    sep = "\t", skip = 1, header = T) %>%
    dplyr::rename(experiment_accession = Accession) %>%
    mutate(file_info = map(experiment_accession, ~ encode_file_info(.x))) %>%
    unnest(file_info) %>% 
    group_by(experiment_accession) %>%
    mutate(rep_number = as.numeric(factor(replicate))) %>%
    unite(sample_id, experiment_accession, rep_number, sep = "_rep") %>%
    dplyr::select(sample_id, accession, Assay.title,
        Biosample.summary, md5sum, paired_end_identifier) %>%
    unite(fastq_file, sample_id, paired_end_identifier, sep = "_read", remove = F) %>%
    mutate(fq_extension = ".fastq.gz") %>%
    unite(fastq_file, fastq_file, fq_extension, sep = "", remove = F) %>%
    unite(original_file, accession, fq_extension, sep = "")

# Rename fastq files
rename_script <- samples %>%
  ungroup() %>%
  dplyr::select(fastq_file, original_file) %>%
  mutate(command = "mv") %>%
  unite(command, command, original_file, fastq_file, sep = " ")

write_lines(c("#!/bin/bash", rename_script$command), "rename.sh")

system("chmod u+x rename.sh")
system("cd fastq; bash ../rename.sh; cd ..")
```

### md5 sum check

``` r
md5 <- samples %>%
  ungroup() %>%
  dplyr::select(md5sum, fastq_file) %>%
  unite(line, md5sum, fastq_file, sep = "  ")

write_lines(md5$line, "fastq/md5.txt")
md5_results <- system("cd fastq; md5sum -c md5.txt; cd ..", intern = TRUE)
write.csv(md5_results, "fastq/md5_check.csv")

# If this check returns TRUE, the md5 check is passed.
all(gsub(".* ", "", md5_results)=="OK")
```

    ## [1] TRUE

### Finalize sample sheet for RNA-seq

``` r
# Rename columns and delete unneeded "original_file" column
samples <- samples %>%
  dplyr::rename(fastq = fastq_file,
                seq_type = Assay.title,
                sample_name = Biosample.summary) %>%
  # The minus sign will remove this column -- which we no longer need.
  dplyr::select(-original_file) 

# Format a samplesheet for DESeq
samplesheet <- samples %>%
  #id_cols is a parameter in pivot wider to select the cols
  # the paired end identifier becomes the "marienette" string of the data-frame.
  # There are two values and thus all the current cols will be split into 2 (one for each pe-id)
  pivot_wider(id_cols = c("sample_id", "seq_type", "sample_name"),
              names_from = paired_end_identifier,
              values_from = c("fastq", "md5sum"))

# Cleaning up sample sheet (removing spaces - re-arrange etc)
samplesheet <- samplesheet %>%
# cleaning up column "sample_name" that has spaces in it to replace with underscore
    mutate(condition = gsub(" ", "_", sample_name) %>% tolower()) %>%
# splitting up "sample_id" to extract replicate number (by "_" )
    separate(sample_id, into = c("experiment_accession", "replicate"), 
        remove = FALSE, sep = "_") %>%
# replicate col values came from sample id and are currently rep1 or rep2
# we want to remove the "rep" with gsub to "R" and iterative using mutate
    mutate(replicate = gsub("rep", "R", replicate)) %>%
# we are writting over the sample_name col and uniting condition and replicate info 
# into the previous sample_name col. syntax: (data frame - implied from tidy, new_col_name, what to unite)
    unite(sample_name, condition, replicate, sep = "_", remove = FALSE)

# FINAL cleanup of col names etc.
samplesheet <- samplesheet %>%
    mutate(cell_type = "hepg2",
        condition = gsub("hepg2_", "", condition)) %>%
    dplyr::select(sample_id, sample_name, replicate, condition,
        cell_type, seq_type, fastq_1, fastq_2, md5sum_1,
        md5sum_2)

write_csv(samplesheet, "samplesheet.csv")
```

### Design file for RNA-seq

``` r
base_path <- "fastq/"

design <- samplesheet %>%  
# Modify 4 cols adding "sample" and "strandedness"
# Add a file path to "fastq_1" and _2 cols
    mutate(sample = gsub(" ", "_", sample_name) %>% tolower(),
        strandedness = "unstranded",
# Paste file path into fastq_1 and _2 cols
        fastq_1 = paste0(base_path, fastq_1),
        fastq_2 = paste0(base_path, fastq_2)) %>%
# Rretrieve replicate number for sample_id and make it just a number 1 or 2 
    separate(sample_id, into = c("experiment_accession", "replicate"), sep = "_", remove = FALSE) %>%
# Don't need "rep"
    mutate(replicate = gsub("rep", "", replicate)) %>%
# Only these columns are needed for DESeq
    dplyr::select(sample, fastq_1, fastq_2, strandedness)

# remove "_r1" and "_r2" from sample column
design <- design %>%
  mutate(sample = gsub("_r1", "", sample),
    sample = gsub("_r2", "", sample))

# Make sure the files exist as listed
all(sapply(design$fastq_1, file.exists))
```

    ## [1] TRUE

``` r
all(sapply(design$fastq_2, file.exists))
```

    ## [1] TRUE

``` r
# Write out for NextFlow
write_csv(design, "design.csv")
write_csv(samples, "samples.csv")
write_csv(samplesheet, "samplesheet.csv")
```

## NextFlow configuration and slurm commands

The design file produced by the above code was fed into nf-core/rnaseq
1.4.2 using the following nextflow.config and run parameters on
Singularity 3.1.1.

### nextflow.config

``` bash
process {
  executor='slurm'
  queue='short'
  memory='16 GB'
  maxForks=10
}
```

### Settings for NextFlow run

``` bash
#!/bin/bash
#SBATCH -p long
#SBATCH --job-name=HEPG2_rna_seq
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=your_email
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --mem=6gb
#SBATCH --time=20:00:00
#SBATCH --output=nextflow.out
#SBATCH --error=nextflow.err

pwd; hostname; date
echo "Here we go You've requested $SLURM_CPUS_ON_NODE core."

module load singularity/3.1.1

nextflow run nf-core/rnaseq -r 1.4.2 \
-resume \
-profile singularity \
--reads 'path/17_API_RNASEQ/fastq/*{_read1,_read2}.fastq.gz' \
--fasta /scratch/Shares/rinnclass/CLASS_2022/data/genomes/GRCh38.p13.genome.fa \
--gtf /scratch/Shares/rinn/genomes/Homo_sapiens/Gencode/v32/gencode.v32.annotation.gtf \
--pseudo_aligner salmon \
--gencode \
--email your_email@colorado.edu \
-c nextflow.config

date
```

### Matching Salmon data to the sample sheet

``` r
salmon_tpm <- read_csv("salmon_merged_gene_tpm.csv")
# Average tpm for each replicate and merge with sample sheet
tpm <- salmon_tpm %>%
    pivot_longer(cols = 2:ncol(.), names_to = "sample_id", values_to = "tpm") %>%
    merge(samplesheet) %>%
    group_by(gene_id, condition) %>%
    summarize(tpm = mean(tpm, na.rm = T)) %>%
    pivot_wider(names_from = condition, values_from = tpm, names_prefix = "tpm_")
```

# Binding Status and Expression

We wanted to understand how expression changes for promoters with
different amounts of TF binding. Here we are looking at all of the
genes, the expression of each one, and how many TFs are bound to its
promoter. We categorized each gene based on “binding status:, first by
low, medium, and high. Low is &gt;100dbps, medium is 100-300, and high
is 300-450 dips. We then looked at this via a density plot of expression
values with the categories denoted by different colors. This gave us
insight into the distribution of the two groups. We found that
typically, things with more binding events are more highly expressed,
but not always. To further analyze this, we split the binding status
into more categories, including “super low” binding, meaning &lt;50dbps,
and “super high” binding, meaning &gt;400 dbps. We found similar
expression between genes that have 0 to 100 genes bound at their
promoters, but much more variability between 100-450 dbps.

``` r
# making high and low binder columns, want to know how binding versus expression change for the low and high binding promoters

#first adding a column called "binding_status" to our promoter_features_df, and will label the promoters as either high, medium or low
promoter_features_df_slice <- read.csv("peak_occurence_dataframe.csv")
promoter_features_df_slice <- merge(promoter_features_df_slice, tpm)
promoter_features_df_slice <- filter(promoter_features_df_slice, promoter_features_df_slice$tpm_homo_sapiens_hepg2 > 0)
promoter_features_df_slice <- promoter_features_df_slice %>%
mutate(binding_status=ifelse(number_of_dbp>300, "high", ifelse(number_of_dbp>=100, "medium", "low")))

# just to get an idea of what's in these categories
table(promoter_features_df_slice$binding_status)
```

    ## 
    ##   high    low medium 
    ##   5563  13118   9745

``` r
# get a 5663 high binders, 20598 low binders, and 10451 medium binders

#plotting the binding vs expression of the categories high, medium, and low
ggplot(promoter_features_df_slice, aes(x= log2(tpm_homo_sapiens_hepg2), color=binding_status)) + geom_density()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/binding%20status%20vs%20expression-1.png)<!-- -->

``` r
#result: more things bound= more expressed, but not always! splitting it into more categories
#making more fl categories- super low is <50, and super high is >400 dbps
promoter_features_df_slice <- promoter_features_df_slice %>%
mutate(binding_status=ifelse(number_of_dbp>400, "superhigh", ifelse(number_of_dbp>300, "high", ifelse(number_of_dbp>100, "medium", ifelse(number_of_dbp>=50, "low", "superlow")))))
ggplot(promoter_features_df_slice, aes(x= log2(tpm_homo_sapiens_hepg2), color=binding_status)) + geom_density()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/binding%20status%20vs%20expression-2.png)<!-- -->

``` r
table(promoter_features_df_slice$binding_status)
```

    ## 
    ##      high       low    medium superhigh  superlow 
    ##      5484      1680      9721        79     11462

``` r
#     high       low    medium superhigh  superlow
#     5581      2133     10417        82     18499
# result is showing that there's a category between low and super low binding there's a similar number that have 100-0 promoter binding events yet are expressed at the same level. However, from 100-450 there is more variation. 
```

# Expression Increases with Number of DBP

``` r
promoter_features_df <- read.csv("peak_occurence_dataframe.csv")
promoter_features_df <- merge(promoter_features_df, tpm)
# We will plot:
# x-axis = number of DBPs bound on the promoter
# y-axis = expression level of that gene.
# y-axis total RNA TPM (+0.0001 to avoid 0 issues)
       # x-axis is number of DBPs
ggplot(promoter_features_df,
            aes(y = log2(tpm_homo_sapiens_hepg2 + 0.001), x = number_of_dbp, color = gene_type)) +
  geom_point(data = promoter_features_df %>% filter(tpm_homo_sapiens_hepg2 < 0.001),
             shape = 17, alpha = 0.7) +
  # Adding a generative additive model
  geom_smooth(method = 'gam', formula = y ~ s(x, bs = "cs")) +
  # this adds the statistics from the gam to the figure
  stat_cor() +
  # this is just to make it look nice.
  scale_x_continuous(expand = c(0,0)) +
  # adding colors manually
  scale_color_manual(values = c("#A8404C", "#424242"), name = "Gene type") +
  # title
  ggtitle("Expression vs. promoter binding events") +
  # labeling axes
  xlab(expression('Number of DBPs')) +
  ylab(expression(log[2](TPM))) +
  theme_paperwhite()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/DBP%20promoter%20binding%20versus%20total%20RNA%20expression-1.png)<!-- -->

# Examining Binding Protein Identity vs. Expression Level

The reservoir class of promoters is surprising given that typically a
promoter with a the capability to bind a large number of DNA binding
proteins experiences high expression levels. We examined the identity of
the DBP binding to reservoirs as well as promoters with high expression
levels to determine if there are DBP distinct to either the reservoirs
or high-transcription promoters.

``` r
high_trans <- filter(promoter_features_df, log2(tpm_homo_sapiens_nuclear_fraction + 0.001)> -5, number_of_dbp > 350)
low_trans <- filter(promoter_features_df, log2(tpm_homo_sapiens_nuclear_fraction + 0.001)< -8, number_of_dbp > 350)

some_trans <- filter(promoter_features_df, log2(tpm_homo_sapiens_nuclear_fraction + 0.001)> -5, 100 < number_of_dbp, number_of_dbp < 300)
no_trans <- filter(promoter_features_df, log2(tpm_homo_sapiens_nuclear_fraction + 0.001)< -8, 100 < number_of_dbp, number_of_dbp < 300)
```

``` r
high_binders <- promoter_peak_occurence[,colSums(promoter_peak_occurence) > 200]

DBP_high_trans <- matrix(NA, nrow = 1)
DBP_low_trans <- matrix(NA, nrow = 1)
DBP_some_trans <- matrix(NA, nrow = 1)
DBP_no_trans <- matrix(NA, nrow = 1)

for (col in 1:ncol(high_binders)) {
  gene = colnames(high_binders)[col]
  if (gene %in% high_trans[,'gene_id']) {
    for (row in 1:nrow(high_binders)){
      if (high_binders[row,col] == 1 && !(rownames(high_binders)[row] %in% DBP_high_trans)){
        DBP_high_trans <- rbind(DBP_high_trans, rownames(high_binders)[row])
      }
    }
  }
  if (gene %in% low_trans[,'gene_id']) {
    for (row in 1:nrow(high_binders)){
      if (high_binders[row,col] == 1 && !(rownames(high_binders)[row] %in% DBP_low_trans)){
        DBP_low_trans <- rbind(DBP_low_trans, rownames(high_binders)[row])
      }
    }
  }
  if (gene %in% some_trans[,'gene_id']) {
    for (row in 1:nrow(high_binders)){
      if (high_binders[row,col] == 1 && !(rownames(high_binders)[row] %in% DBP_some_trans)){
        DBP_some_trans <- rbind(DBP_some_trans, rownames(high_binders)[row])
      }
    }
  }
  if (gene %in% no_trans[,'gene_id']) {
    for (row in 1:nrow(high_binders)){
      if (high_binders[row,col] == 1 && !(rownames(high_binders)[row] %in% DBP_no_trans)){
        DBP_no_trans <- rbind(DBP_no_trans, rownames(high_binders)[row])
      }
    }
  }
}
DBP_high_trans <- DBP_high_trans[-1,]
DBP_low_trans <- DBP_low_trans[-1,]
DBP_some_trans <- DBP_some_trans[-1,]
DBP_no_trans <- DBP_no_trans[-1,]
```

``` r
DBP_low_only <- matrix(NA, nrow = 1)
DBP_high_only <- matrix(NA, nrow = 1)

for (dbp in DBP_high_trans){
  #Identify different DBP in subset for high and low transcription
  if(!(dbp %in% DBP_low_trans)){
    DBP_high_only <- rbind(DBP_high_only, dbp)
  }
  
}

for (dbp in DBP_low_trans){
  if(!(dbp %in% DBP_high_trans)){
    DBP_low_only <- rbind(DBP_low_only, dbp)
  }
}

DBP_high_only <- DBP_high_only[-1,]
DBP_low_only <- DBP_low_only[-1,]

print('DBP which bind exclusively to high transcription promoters:')
```

    ## [1] "DBP which bind exclusively to high transcription promoters:"

``` r
print(DBP_high_only)
```

    ##       dbp       dbp       dbp       dbp       dbp 
    ##    "MBD4"  "TCF7L2"   "TCF12"   "KAT2B" "H3K9me3"

``` r
DBP_some_only <- matrix(NA, nrow = 1)
DBP_no_only <- matrix(NA, nrow = 1)

for (dbp in DBP_some_trans){
  #Identify different DBP in subset for high and low transcription
  if(!(dbp %in% DBP_no_trans)){
    DBP_some_only <- rbind(DBP_some_only, dbp)
  }
}

for (dbp in DBP_no_trans){
  if(!(dbp %in% DBP_some_trans)){
    DBP_no_only <- rbind(DBP_no_only, dbp)
  }
}

DBP_some_only <- DBP_some_only[-1,]
DBP_no_only <- DBP_no_only[-1,]

print('DBP which bind exclusively to promoters with medium transcription:')
```

    ## [1] "DBP which bind exclusively to promoters with medium transcription:"

``` r
print(DBP_some_only)
```

    ##        dbp        dbp        dbp        dbp        dbp        dbp 
    ##    "KAT2B"   "ZNF367"     "BRD4"    "TCF12" "H3K36me3"     "MBD4"

We discovered no DBP unique to reservoirs; however, several DBP were
found to bind only to promoters with transcription on. These proteins do
not bind to reservoirs despite nearly 400 DBP which can bind to either
reservoir or non-reservoir promoters suggesting. Of particular interest
is MBD4 which is a repressor but only binds to promoters for highly
transcribed genes. Overall this analysis shows that reservoirs and
non-reservoir promoters are either distinguished by relatively small
differences in selectivity towards DBP or that the distinguishing
repressor is not included in this set of 460 DBPs. Further insight could
be provided by examining the relative binding of the 460 DBP to these
classes of promoters to eliminate proteins that rarely bind these
promoters from analysis and by increasing the library of DBP for
analysis.

# Meta Plots of Non-Reservoir Promoters

Looking deeper into the unique DBPs that do not bind to reservoirs but
are highly expressed. The meta plots can elucidate any differences in
promoter window binding patterns.

## KAT2B

``` r
# Indexing gencode_gr to grab "genes"
#genes <- gencode_gr[gencode_gr$type == "gene"]
# lncRNA promoter profiles
#lncrna_genes <- genes[genes$gene_type == "lncRNA"]
#lncrna_promoters <- promoters(lncrna_genes, upstream = 2.5e3, downstream = 2.5e3)

lncrna_metaplot_profile_KAT2B <- profile_tss(filtered_consensus_peaks[["KAT2B"]], lncrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# mRNA promoter profiles
#mrna_genes <- genes[genes$gene_type == "protein_coding"]
#mrna_promoters <- promoters(mrna_genes, upstream = 2.5e3, downstream = 2.5e3)

mrna_metaplot_profile_KAT2B <- profile_tss(filtered_consensus_peaks[["KAT2B"]], mrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# We can row bind these dataframes so that we can plot them on the same plot
mrna_metaplot_profile_KAT2B$gene_type <- "mRNA"
lncrna_metaplot_profile_KAT2B$gene_type <- "lncRNA"
combined_metaplot_profile_KAT2B <- bind_rows(mrna_metaplot_profile_KAT2B, lncrna_metaplot_profile_KAT2B)

ggplot(combined_metaplot_profile_KAT2B, 
       aes(x = x, y = dens, color = gene_type)) +
  geom_vline(xintercept = 0, lty = 2) + 
  geom_line(size = 1.5) + 
  ggtitle("KAT2B Promoter Metaplot") + 
  scale_x_continuous(breaks = c(-2500, 0, 2500),
                     labels = c("-2.5kb", "TSS", "+2.5kb"),
                     name = "") + 
  ylab("Peak frequency") + 
  scale_color_manual(values = c("#424242","#a8404c")) +
  theme_paperwhite()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/metaplot%20mrna%20lncrna%20KAT2B-1.png)<!-- -->

## TCF7L2

``` r
# Indexing gencode_gr to grab "genes"
#genes <- gencode_gr[gencode_gr$type == "gene"]
# lncRNA promoter profiles
#lncrna_genes <- genes[genes$gene_type == "lncRNA"]
#lncrna_promoters <- promoters(lncrna_genes, upstream = 2.5e3, downstream = 2.5e3)

lncrna_metaplot_profile_TCF7L2 <- profile_tss(filtered_consensus_peaks[["TCF7L2"]], lncrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# mRNA promoter profiles
#mrna_genes <- genes[genes$gene_type == "protein_coding"]
#mrna_promoters <- promoters(mrna_genes, upstream = 2.5e3, downstream = 2.5e3)

mrna_metaplot_profile_TCF7L2 <- profile_tss(filtered_consensus_peaks[["TCF7L2"]], mrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# We can row bind these dataframes so that we can plot them on the same plot
mrna_metaplot_profile_TCF7L2$gene_type <- "mRNA"
lncrna_metaplot_profile_TCF7L2$gene_type <- "lncRNA"
combined_metaplot_profile_TCF7L2 <- bind_rows(mrna_metaplot_profile_TCF7L2, lncrna_metaplot_profile_TCF7L2)

ggplot(combined_metaplot_profile_TCF7L2, 
       aes(x = x, y = dens, color = gene_type)) +
  geom_vline(xintercept = 0, lty = 2) + 
  geom_line(size = 1.5) + 
  ggtitle("TCF7L2 Promoter Metaplot") + 
  scale_x_continuous(breaks = c(-2500, 0, 2500),
                     labels = c("-2.5kb", "TSS", "+2.5kb"),
                     name = "") + 
  ylab("Peak frequency") + 
  scale_color_manual(values = c("#424242","#a8404c")) +
  theme_paperwhite()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/metaplot%20mrna%20lncrna%20TCF7L2-1.png)<!-- -->

## TCF12

``` r
# Indexing gencode_gr to grab "genes"
#genes <- gencode_gr[gencode_gr$type == "gene"]
# lncRNA promoter profiles
#lncrna_genes <- genes[genes$gene_type == "lncRNA"]
#lncrna_promoters <- promoters(lncrna_genes, upstream = 2.5e3, downstream = 2.5e3)

lncrna_metaplot_profile_TCF12 <- profile_tss(filtered_consensus_peaks[["TCF12"]], lncrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# mRNA promoter profiles
#mrna_genes <- genes[genes$gene_type == "protein_coding"]
#mrna_promoters <- promoters(mrna_genes, upstream = 2.5e3, downstream = 2.5e3)

mrna_metaplot_profile_TCF12 <- profile_tss(filtered_consensus_peaks[["TCF12"]], mrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# We can row bind these dataframes so that we can plot them on the same plot
mrna_metaplot_profile_TCF12$gene_type <- "mRNA"
lncrna_metaplot_profile_TCF12$gene_type <- "lncRNA"
combined_metaplot_profile_TCF12 <- bind_rows(mrna_metaplot_profile_TCF12, lncrna_metaplot_profile_TCF12)

ggplot(combined_metaplot_profile_TCF12, 
       aes(x = x, y = dens, color = gene_type)) +
  geom_vline(xintercept = 0, lty = 2) + 
  geom_line(size = 1.5) + 
  ggtitle("TCF12 Promoter Metaplot") + 
  scale_x_continuous(breaks = c(-2500, 0, 2500),
                     labels = c("-2.5kb", "TSS", "+2.5kb"),
                     name = "") + 
  ylab("Peak frequency") + 
  scale_color_manual(values = c("#424242","#a8404c")) +
  theme_paperwhite()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/metaplot%20mrna%20lncrna%20TCF12-1.png)<!-- -->

## H3K9me3

``` r
# Indexing gencode_gr to grab "genes"
#genes <- gencode_gr[gencode_gr$type == "gene"]
# lncRNA promoter profiles
#lncrna_genes <- genes[genes$gene_type == "lncRNA"]
#lncrna_promoters <- promoters(lncrna_genes, upstream = 2.5e3, downstream = 2.5e3)

lncrna_metaplot_profile_H3K9me3 <- profile_tss(filtered_consensus_peaks[["H3K9me3"]], lncrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# mRNA promoter profiles
#mrna_genes <- genes[genes$gene_type == "protein_coding"]
#mrna_promoters <- promoters(mrna_genes, upstream = 2.5e3, downstream = 2.5e3)

mrna_metaplot_profile_H3K9me3 <- profile_tss(filtered_consensus_peaks[["H3K9me3"]], mrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# We can row bind these dataframes so that we can plot them on the same plot
mrna_metaplot_profile_H3K9me3$gene_type <- "mRNA"
lncrna_metaplot_profile_H3K9me3$gene_type <- "lncRNA"
combined_metaplot_profile_H3K9me3 <- bind_rows(mrna_metaplot_profile_H3K9me3, lncrna_metaplot_profile_H3K9me3)

ggplot(combined_metaplot_profile_H3K9me3, 
       aes(x = x, y = dens, color = gene_type)) +
  geom_vline(xintercept = 0, lty = 2) + 
  geom_line(size = 1.5) + 
  ggtitle("H3K9me3 Promoter Metaplot") + 
  scale_x_continuous(breaks = c(-2500, 0, 2500),
                     labels = c("-2.5kb", "TSS", "+2.5kb"),
                     name = "") + 
  ylab("Peak frequency") + 
  scale_color_manual(values = c("#424242","#a8404c")) +
  theme_paperwhite()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/metaplot%20mrna%20lncrna%20H3K9me3-1.png)<!-- -->

KAT2B had a small peak of mRNA binding and well spread in the window
while lncRNA had a large expected peak around TSS and -2.5kb. TCF7L2 and
TCF12 had similar binding patterns were a large peak for both mRNA and
lncRNA were shown around the TSS. H3K9me3 had a unique binding pattern,
especially for mRNA where the largest peak was around -2.3kb and two
lower peaks around the TSS and +2.5kb sites. lncRNA had a single peak
around TSS.

As more analysis is done to categorize the DBP in and outside of
reservoirs, meta plots can be used to find binding patterns unique to
these groups. More analysis can also be done on these metaplots to
uncover patterns that are RNA specific or a certain type of RNA (lncRNA
or mRNA).

# Some genes are expressed much more than others with a similar number of DBP binding events

``` r
low_peak_occurence_df <- peak_occurence_df %>% filter(peak_occurence_df$number_of_dbp < 160)
high_peak_occurence_df <- peak_occurence_df %>% filter(peak_occurence_df$number_of_dbp >= 160)

dbp_tpm <- merge(peak_occurence_df, tpm, by = "gene_id")
low_binders_dbp_tpm <- merge(low_peak_occurence_df, tpm, by = "gene_id")
high_binders_dbp_tpm <- merge(high_peak_occurence_df, tpm, by = "gene_id")

# Removing irrelevant information
dbp_tpm_sm <- subset(dbp_tpm, select = c(gene_id, gene_name, gene_type, number_of_dbp, tpm_homo_sapiens_hepg2, tpm_homo_sapiens_cytosolic_fraction, tpm_homo_sapiens_insoluble_cytoplasmic_fraction, tpm_homo_sapiens_nuclear_fraction, tpm_homo_sapiens_membrane_fraction))

low_binders_sm <- subset(low_binders_dbp_tpm, select = c(gene_id, gene_name, gene_type, number_of_dbp, tpm_homo_sapiens_hepg2, tpm_homo_sapiens_cytosolic_fraction, tpm_homo_sapiens_insoluble_cytoplasmic_fraction, tpm_homo_sapiens_nuclear_fraction, tpm_homo_sapiens_membrane_fraction))

high_binders_sm <- subset(high_binders_dbp_tpm, select = c(gene_id, gene_name, gene_type, number_of_dbp, tpm_homo_sapiens_hepg2, tpm_homo_sapiens_cytosolic_fraction, tpm_homo_sapiens_insoluble_cytoplasmic_fraction, tpm_homo_sapiens_nuclear_fraction, tpm_homo_sapiens_membrane_fraction))
```

## Expression and binding of lncRNAs and mRNAs

Though both lncRNAs and mRNAs are observed in both the lower and higher
peaks of the bimodal distribution described above, protein coding genes
are more likely than are lncRNAs to bind many DBPs.

``` r
# TODO find a way to combine these plots into one.
#barplot(table(dbp_tpm$gene_type), ylim = c(0, 20000), main = "Total Number of Genes in Sample by Gene Type")
#barplot(table(low_binders_dbp_tpm$gene_type), ylim = c(0, 20000), main = "Number of Genes in Sample by Gene Type\nwith Fewer Than 160 DBP")
#barplot(table(high_binders_dbp_tpm$gene_type), ylim = c(0, 20000), main = "Number of Genes in Sample by Gene Type\nwith More Than 160 DBP")
# TODO I give up. Here's what I was trying to do:
```

![Number of Genes in Sample by Gene
Type](analysis/figures/total_genes_barplot.png)

## Outliers

Some genes were expressed much more than genes with a similar number of
binding proteins. An arbitrary threshold to help identify possible genes
of interest was defined with the curve below.

``` r
# This parabola is the (arbitrary) cutoff for outliers
ggplot(dbp_tpm, aes(number_of_dbp, tpm_homo_sapiens_hepg2)) +
  geom_count() +
  stat_function(fun = function(x){0.05*(x-175)^2 + 2500}, color = "red") +
  ylim(0,10000) +
  theme_minimal()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/DBP%20to%20TPM%20outlier%20definition-1.png)<!-- -->

``` r
# TODO find a way to overlay every fraction.
```

``` r
# Preparing for three plots of TPM ~ DBP:
# 1) all binders
ab <- dbp_tpm_sm %>% pivot_longer(!c(gene_id, gene_name, gene_type, number_of_dbp), names_to = "fraction", values_to = "tpm") %>% filter(tpm > 0)
# 2) low binders
lb <- low_binders_sm %>% pivot_longer(!c(gene_id, gene_name, gene_type, number_of_dbp), names_to = "fraction", values_to = "tpm") %>% filter(tpm > 0)
# 3) high binders
hb <- high_binders_sm %>% pivot_longer(!c(gene_id, gene_name, gene_type, number_of_dbp), names_to = "fraction", values_to = "tpm") %>% filter(tpm > 0)
```

All outliers are high binders, except ALB (albumin) in some fractions.

``` r
# List of outliers
outsab <- ab %>% filter((tpm - 0.05*(number_of_dbp-175)^2) > 2500)
# print
outsab$gene_name
```

    ##  [1] AFP        AFP        FTL        FTL        RPS19      RPS12     
    ##  [7] EEF1A1     EEF1A1     EEF1A1     ALB        ALB        ALB       
    ## [13] HIST1H4J   HIST1H4J   AL355075.4 RMRP      
    ## 36761 Levels: A1BG A1BG-AS1 A1CF A2M A2M-AS1 A2ML1 A2ML1-AS1 A2ML1-AS2 ... ZZEF1

``` r
# List of outliers in low binders
outslb <- lb %>% filter((tpm - 0.05*(number_of_dbp-175)^2) > 2500)
# print
outslb$gene_name
```

    ## [1] ALB ALB ALB
    ## 36761 Levels: A1BG A1BG-AS1 A1CF A2M A2M-AS1 A2ML1 A2ML1-AS1 A2ML1-AS2 ... ZZEF1

``` r
# List of outliers in high binders
outshb <- hb %>% filter((tpm - 0.05*(number_of_dbp-175)^2) > 2500)
#print
outshb$gene_name
```

    ##  [1] AFP        AFP        FTL        FTL        RPS19      RPS12     
    ##  [7] EEF1A1     EEF1A1     EEF1A1     HIST1H4J   HIST1H4J   AL355075.4
    ## [13] RMRP      
    ## 36761 Levels: A1BG A1BG-AS1 A1CF A2M A2M-AS1 A2ML1 A2ML1-AS1 A2ML1-AS2 ... ZZEF1

Of the outliers, only two are lncRNAs. One of those happens to be RMRP,
the RNA component of the mitochondrial RNA-processing endoribonuclease.

``` r
# All genes
ggplot(ab, aes(number_of_dbp, log10(tpm + 1), color = gene_type)) +
  ggtitle("Log(TPM) vs. Number of DBP in HepG2 Fractions") +
  xlab("Number of DBPs") + ylab("log( tpm + 1 )") +
  geom_point(data=outsab, x=outsab$number_of_dbp, y=log10(outsab$tpm + 1), color="tomato", shape = 10, size = 2, alpha = 0.75) +
  geom_text_repel(data = outsab, aes(number_of_dbp, log10(tpm + 1), label = gene_name)) +
  geom_jitter(alpha = 0.3) +
  facet_wrap(~fraction) +
  theme_minimal()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/DBP%20to%20TPM%20plots-1.png)<!-- -->

``` r
ggsave("analysis/figures/logtpm_dbp_total.pdf", units = "in", width = 16, height = 10)

# Low binders
ggplot(lb, aes(number_of_dbp, log10(tpm + 1), color = gene_type)) +
  ggtitle("Log(TPM) vs. Number of DBP in HepG2 Fractions (low binders < 160 DBP)") +
  xlab("Number of DBPs") + ylab("log( tpm + 1 )") +
  geom_point(data=outslb, x=outslb$number_of_dbp, y=log10(outslb$tpm + 1), color="tomato", shape = 10, size = 2, alpha = 0.75) +
  geom_text_repel(data = outslb, aes(number_of_dbp, log10(tpm + 1), label = gene_name)) +
  geom_jitter(alpha = 0.3) +
  facet_wrap(~fraction) +
  theme_minimal()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/DBP%20to%20TPM%20plots-2.png)<!-- -->

``` r
ggsave("analysis/figures/logtpm_dbp_low_binders.pdf", units = "in", width = 16, height = 10)

# High binders
ggplot(hb, aes(number_of_dbp, log10(tpm + 1), color = gene_type)) +
  ggtitle("Log(TPM) vs. Number of DBP in HepG2 Fractions (high binders > 160 DBP)") +
  xlab("Number of DBPs") + ylab("log( tpm + 1 )") +
  geom_point(data=outshb, x=outshb$number_of_dbp, y=log10(outshb$tpm + 1), color="tomato", shape = 10, size = 2, alpha = 0.75) +
  geom_text_repel(data = outshb, aes(number_of_dbp, log10(tpm + 1), label = gene_name)) +
  geom_jitter(alpha = 0.3) +
  facet_wrap(~fraction) +
  xlim(160, 430) +
  theme_minimal()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/DBP%20to%20TPM%20plots-3.png)<!-- -->

``` r
ggsave("analysis/figures/logtpm_dbp_high_binders.pdf", units = "in", width = 16, height = 10)
```

## DBP

LOL

``` r
# Indexing gencode_gr to grab "genes"
#genes <- gencode_gr[gencode_gr$type == "gene"]
# lncRNA promoter profiles
#lncrna_genes <- genes[genes$gene_type == "lncRNA"]
#lncrna_promoters <- promoters(lncrna_genes, upstream = 2.5e3, downstream = 2.5e3)

lncrna_metaplot_profile_DBP <- profile_tss(filtered_consensus_peaks[["DBP"]], lncrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# mRNA promoter profiles
#mrna_genes <- genes[genes$gene_type == "protein_coding"]
#mrna_promoters <- promoters(mrna_genes, upstream = 2.5e3, downstream = 2.5e3)

mrna_metaplot_profile_DBP <- profile_tss(filtered_consensus_peaks[["DBP"]], mrna_promoters, upstream = 2.5e3, downstream = 2.5e3)

# We can row bind these dataframes so that we can plot them on the same plot
mrna_metaplot_profile_DBP$gene_type <- "mRNA"
lncrna_metaplot_profile_DBP$gene_type <- "lncRNA"
combined_metaplot_profile_DBP <- bind_rows(mrna_metaplot_profile_DBP, lncrna_metaplot_profile_DBP)

ggplot(combined_metaplot_profile_DBP, 
       aes(x = x, y = dens, color = gene_type)) +
  geom_vline(xintercept = 0, lty = 2) + 
  geom_line(size = 1.5) + 
  ggtitle("DBP Promoter Metaplot") + 
  scale_x_continuous(breaks = c(-2500, 0, 2500),
                     labels = c("-2.5kb", "TSS", "+2.5kb"),
                     name = "") + 
  ylab("Peak frequency") + 
  scale_color_manual(values = c("#424242","#a8404c")) +
  theme_paperwhite()
```

![](genome_wide_dna_binding_hepg2_files/figure-gfm/metaplot%20mrna%20lncrna%20DBP-1.png)<!-- -->

Amusing as it is to at least one person to investigate a DBP called
“DBP”, it happens to be relevant. DBP is D-Box Binding PAR BZIP
Transcription Factor, which binds the “D-Box” sequence
(5’-RTTAYGTAAY-3’) found in the ALB gene promoter.

### Anyway…

These outliers raise an important question:

## How Meaningful are HepG2 or Other Cancer Cell Data for Healthy Cells?

HepG2 cells were isolated from the liver cancer cells of a 15 year old
Caucasian male [Aden 1979](https://doi.org/10.1038/282615a0). These
cells excrete large quantities of albumin, which happens to be one of
the outliers.

One of the outliers is AFP, or alpha fetoprotein. AFP is known to be
diagnostic for hepatoblastoma [Sharma 2017](https://doi.org/10.1053/j.semdp.2016.12.015), and is excreted in
large quantities in HepG2 cells [Aden
1979](https://doi.org/10.1038/282615a0).

The cells were epithelial, and retained the ability to excrete these
plasma proteins, which is a significant reason they were of interest to
researchers in the first place [Aden
1979](https://doi.org/10.1038%2F282615a0).

The next outlier is FTL, the light chain of ferritin. Ferritin is the
main iron storing complex in humans. The presence of elevated serum
ferritin is also diagnostic of cancer [Ahmed
2013](https://doi.org/10.1016/j.bbcan.2013.07.002), unsurprising given
its established role in angiogenesis [Coffman
2009](https://doi.org/10.1073/pnas.0812010106).

The ribosomal protein genes RPS19 and RPS12 are components of the small
ribosomal subunit, and their high expression in the membrane fraction is
unsurprising in any cell. EEF1A1 is another translation protein, and
HIST1H4J is a histone.

This leaves the two lncRNAs, RMRP and AL355075.4. Both of these are
ribonuclease components, and AL355075.4 is antisense to PARP2. RMRP has
been implicated in carcinogenesis [Shao
2016](https://doi.org/10.18632/oncotarget.9336).

A common mistake made in biology is the use of unusual model systems to
draw conclusions about more typical systems, e.g. the use of tumor
supressor gene knockout mice to assay the carcinogenicity of a chemical.
It’s important that we avoid making the same mistake, and these data
would be more meaningful if compared in a range of human cell lines.
