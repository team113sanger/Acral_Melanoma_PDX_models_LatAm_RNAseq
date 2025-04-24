# Detailed information about data analysis process
In the following text I'm going to show the commands and steps I took to do the analysis of the RNAseq data. 
This process assumes that **all the dependencies and required software has been installed**. 
**All  of the required folders will be downloaded with the clone of this repository** 
This folder will become the **_`Project directory`_**
all remaining folders will be created within this location.

_**Note:** All commands will be assumed to be run from inside the_ **_`Project directory`_**


## 1. Creation  of sequencing data manifest from Sanger's internal iRODS database
The first step is to get all the information about the project samples and their sequencing information. This will be retrieved from iRODS database using the **Project ID `6352`**. 

_**NOTE:** iRODS `iinit` needs to be enabled_

We start by defining which is the project directory (_i.e. the full path to folder of this repository was cloned into_):
``` bash
PROJECTDIR=/My/project/full_path/6352_rat_TCR_hepatocellular_carcinoma_RNAseq
```
Then we enable **`iRODS`** and run the script **`Build_manifest_from_irods_cram_information.R`** with the parameters `--seqscape_proj_id 6352 --outdir `
``` bash
PROJECTDIR=/My/project/full_path/6352_rat_TCR_hepatocellular_carcinoma_RNAseq
iinit
#To call the script to build the manifests with project ID Name
module load R/4.1.0
/software/R-4.1.0/bin/Rscript $PROJECTDIR/scripts/Build_manifest_from_irods_cram_information.R --seqscape_proj_id 6352 --outdir $PROJECTDIR/manifests
```

This will create a tab separated table with all the project's information called `6352_cram_manifest_INFO_from_iRODS.txt`
and a file with all the samples names called `6352_manifest_uniq_sup_sample_names.txt`.

The manifest has the following information per sample:

| sample | sample_supplier_name | sample_donor_id | sample_accession_number | study_accession_number | sample_common_name | md5 | id_run | lane | is_paired_read | tag_index | library_id | total_reads | study_id | study_title | reference | library_type | cram | cram_irods_location |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ |

## 2. Generate the commands for CRAM conversion to fastq, alignment and read counting and run each process.
After obtaining the infromation from the sequencing database and compiling it into a project manifest. I then ran the script **`cramtofastq_STAR_RAT_mapping_hseqcount_RSeQC_from_iRODs_based_cram_manifest.R`** with the parameters `--manifest 6352_cram_manifest_INFO_from_iRODS.txt --projectdir $PROJECTDIR`. 

**NOTE:** this script has hardcoded the full path to the following software: `samtools 1.13, STAR v2.7.9a, htseq-counts 0.13.5, RSeQC v4.0.0.0` As well, as the required references.

To run the script I used the following commands:
``` bash 
PROJECTDIR=/My/project/full_path/6352_rat_TCR_hepatocellular_carcinoma_RNAseq
#This script takes the cram manifest and generates the SH file with the farm jobs to import and transform to fastq all of the cram files from iRODs
module load R/4.1.0
/software/R-4.1.0/bin/Rscript $PROJECTDIR/scripts/cramtofastq_STAR_RAT_mapping_hseqcount_RSeQC_from_iRODs_based_cram_manifest.R --manifest 6352_cram_manifest_INFO_from_iRODS.txt --projectdir $PROJECTDIR
```

This script will create several directories:

1. **fastqs:** This folder will contain the `gzip` compressed fastq files from each sample
2. **logs:** This folder contains the logs of the jobs afters their submission
3. **STAR_2_7_9a_bams:** This is the folder for the BAM files mapped with STAR
4. **htseq_counts:** This folder contains counts generated with htsetq-coun
5. **qc_plots:** This folder contains the folders with the quality check qc_plots

This script will also creat a set of different shell (bash) scripts to run each step the process:

1. **`cramtofastq_from_iRODs_jobs.sh`** This script with the commands to submit the jobs for the CRAM import, FASTQ conversion and compression
2. **`cramtofastq_from_iRODs_mapping_jobs.sh:`** This script with the commands to submit the STAR alignment jobs
3. **`cramtofastq_from_iRODs_mapping_counts_jobs.sh:`** This script with the commands to submit the jobs for the read counting using htseq-count
4. **`run_RSeQC_read_NVCpy.sh and run_RSeQC_read_qualpy.sh`** These scripts submit the jobs to check the Nucleotide compositional bias per read  and the read mapping quality metris for each sample using RSeQC.

First to run the  script **`cramtofastq_from_iRODs_jobs.sh`**  I used the following command:
``` shell
iinit 
/bin/sh $PROJECTDIR/scripts/cramtofastq_from_iRODs_jobs.sh
```
This will submit a set of jobs with using the a pipe of commands to import the CRAM, convert it and compress it with gzip. An example of this command is
``` shell
PROJECTDIR=/My/project/full_path/6352_rat_TCR_hepatocellular_carcinoma_RNAseq
iget /seq/illumina/runs/38/38900/lane1/plex1/38900_1#1.cram - | /samtools-1.13/bin/samtools collate -u --threads 6 -O - | /samtools-1.13/bin/samtools fastq --threads 6 -1 $PROJECTDIR/fastqs/6352STDY10233930_R1.fastq.gz -2 $PROJECTDIR/fastqs/6352STDY10233930_R2.fastq.gz -n 
```

Once the CRAM files were converted into FASTQs I ran the  script **`cramtofastq_from_iRODs_mapping_jobs.sh`** to perform the alignments. I used the following command:
``` shell
PROJECTDIR=/My/project/full_path/6352_rat_TCR_hepatocellular_carcinoma_RNAseq
/bin/sh $PROJECTDIR/scripts/cramtofastq_from_iRODs_mapping_jobs.sh
```
This aforementioned script was used to submit a set of jobs for the alignment of each sample against the Rat feference genome using STAR. All jobs requested a computer with 12 threads and 32Gb of RAM. An example of the commands for alignment is
``` shell
PROJECTDIR=/My/project/full_path/6352_rat_TCR_hepatocellular_carcinoma_RNAseq
/STAR-2.7.9a/bin/Linux_x86_64/STAR --genomeDir /lustre/scratch119/casm/team113da/references/rat/Rnor6/STAR_2.7.9a_indexes/STAR_2.7.9a_ERCC92_rlen100_ENSv104 --outFileNamePrefix $PROJECTDIR/STAR2_7_9a_bams/6352STDY10233930/6352STDY10233930_ --runThreadN 12  --readFilesIn $PROJECTDIR/fastqs/6352STDY10233930_R1.fastq.gz $PROJECTDIR/fastqs/6352STDY10233930_R2.fastq.gz --sjdbGTFfile $PROJECTDIR/reference/Rattus_norvegicus.Rnor_6.0.ensv104_ERCC92.gtf --readFilesCommand zcat --outSAMtype BAM SortedByCoordinate --outSAMattrRGline ID:38900_1#1 LB:42564303 SM:RR0065b CN:SC PL:ILLUMINA --outFilterType BySJout --outFilterMultimapNmax 1 --alignSJoverhangMin 8 --alignSJDBoverhangMin 1 --outFilterMismatchNmax 999 --outFilterMismatchNoverReadLmax 0.04 --alignIntronMin 20 --alignIntronMax 1000000 --alignMatesGapMax 1000000 --outSAMattrIHstart 0 --outSAMmultNmax 1 --outSAMstrandField intronMotif --quantMode GeneCounts 
```

After all the mapping jobs have finishes I ran the  script **`cramtofastq_from_iRODs_mapping_counts_jobs.sh`** . To do this I used the following command:
``` shell
PROJECTDIR=/My/project/full_path/6352_rat_TCR_hepatocellular_carcinoma_RNAseq
/bin/sh $PROJECTDIR/scripts/cramtofastq_from_iRODs_mapping_counts_jobs.sh
```
This script used htseq-count to count the number of uniquely mapped reads per gene from the samples' BAM files. All jobs requested a computer with 6 threads and 6Gb of RAM. The parameters for the counting used the counting model "intersection-nonempty", for reference of what this entails see the following [link](https://htseq.readthedocs.io/en/master/count.html?highlight=htseq-count). The `--stranded reverse` option was selected because the libraries were made using Illumina stranded protocol. An example of the commands for counting is shown below:
``` shell
PROJECTDIR=/My/project/full_path/6352_rat_TCR_hepatocellular_carcinoma_RNAseq
/python/bin/htseq-count --format bam --order pos -m intersection-nonempty --stranded reverse --idattr gene_id --type exon $PROJECTDIR/STAR2_7_9a_bams/6352STDY10233930/6352STDY10233930_Aligned.sortedByCoord.out.bam $PROJECTDIR/reference/Rattus_norvegicus.Rnor_6.0.ensv104_ERCC92.gtf >$PROJECTDIR/htseq_counts/6352STDY10233930_htseq_counts_srverse.txt
```

Finally, to run the  scripts **`run_RSeQC_read_NVCpy.sh and run_RSeQC_read_qualpy.sh`**  I used the following commands:
``` shell
PROJECTDIR=/My/project/full_path/6352_rat_TCR_hepatocellular_carcinoma_RNAseq
/bin/sh $PROJECTDIR/scripts/run_RSeQC_read_NVCpy.sh
/bin/sh $PROJECTDIR/scripts/run_RSeQC_read_qualpy.sh
```
These scripts use RSeQC to calculate the reads nucleotide compositional bias per sample and the base quality

## 3. Post porcessing: compile the counting tables, calcutate the Trasncript Per Million (TPM) metrics per sample, sample quality checks 

To perform the post processing analysis of this project's count data I ran the script **`6352_study_count_matrix_prelim_analysis.R`** 


_**NOTE:**_ If you require to install the packages that the srcipt will need, run within _**R-4.1.0**_ :
``` R
BiocManager::install(c("biomaRt", "ggplot2", "limma", "gplots", "RColorBrewer", "gplots", "EDASeq",
                       "tidyr", 
                       "reshape2"), version = '3.13' )
install.packages(c("data.table", "doParallel", "pheatmap"),repos = "https://cran.ma.imperial.ac.uk/")
```

This scripts will generate a series of files, PDFs and a directory within `$PROJECTDIR/results`  these will be named with the prefix `6352_rat_TCR_hepatocellular_carcinoma_RNAseq`. The files and folders produced are :

- **`$PROJECTDIR/results/counts_qc`** This folder contains the plots of the analysis of counts obtained per samples:
    + It contains the following plots:
        * `_RNAseq_RIN_VS_read_counts.pdf` Shows the number of reads counted against the RIN number of the RNA input for each sample
        * `_reads_summary_boxplot.pdf` Shows the total number of reads within the BAM files VS the total number of reads counted
        * `_percent_uniq_mapped_Reads_boxplot.pdf` Contains the precentage of uniquely mapped reads per sample
        * `_data_HTseq_count_summary_by_category_per_sample.pdf` Shows the total number of counts per samply by HTSeq-counts category
    + `6352_rat_TCR_hepatocellular_carcinoma_RNAseqHTseq_counts_summary_by_class.txt` Is a table that contains the count statitstics for all the samples in the cohort by htseq-count class

| sample | gene_read_pair_counts | no_feature_reads | ambiguous_reads | fail_counts |
| ------ | ------ | ------ | ------ | ------ |
| RR0065b | 60700306 | 9828016 | 407405 | NA |
<br>
   
- **`_HTSeq_Sreverse_STAR_ENSv104_ERCC_TPM_ENS_rgd_symbol_IDS.txt`** Is a table that contains the _**TPM**_ transformed counts for all the ENSMBLv104 + ERCCv92 genes  having samples by column and genes per row. The first two columns contain the **external_gene_game** (i.e. the gene symbol) and the second clumn the **ESNEMBL_gene_ID**
- **`_HTSeq_Sreverse_STAR_ENSv104_ERCC_RAW_frag_counts_sample_sanger_IDs.tsv`** Is a table that contains the **_RAW_** read counts for all the ENSMBLv104 + ERCCv92 genes  having samples by column (**using sanger's sample ID**) and genes per row (**using ensembl_gene_ID**) as the gene names. The first columns contains the **ensembl_gene_name** 
- **`_HTSeq_Sreverse_STAR_ENSv104_ERCC_RAW_frag_counts_sample_supplier_name_IDs.tsv`** It is a table that contains the same information as above but using the **supplier_sample_names** as sample names on each column
- **`_HTSeq_Sreverse_STAR_ERCC_ONLY_RAW_frag_counts_sample_sanger_IDs.tsv`** It is a table that contains the same information as above but just for the ERCCv92 spike in sequences 
- **`_STAR_mapping_statisics.txt`** This table contains all the mapping summary statistics for all the sample of the entire cohort

In additon to the files mentioned above the folder also contains the following files with information about the **reference and the R versions** of both R and the packages used. These files are : 

- **`Analysis_R_session_INFO.txt`** Contains the R-base version and packages versions used for the analyis and graphs. 
- **`ENSEMBL_v104_Rat_gene_biotypes_full_gene_information.txt`** Contains useful metadata information for all the ENSEMBL_gene_IDs used in the annotation such as RGID symbols, gene_biotypes, chromosomal coordinates of the gene. 
- **`ENSEMBL_v104_Rat_genes_ERCCv92_full_gene_length_information.txt`**  Contains the gene length in bases used to do the  TPM transformation. The information was obtain from ENSEMBL's v104 dabase _biomaRt_ using `EDASeq::getGeneLengthAndGCContent` function
- **`ENSEMBL_v104_Rat_genes_full_gene_length_GCcontent_information.txt`**  Contains the gene length in bases  and GC content information per gene for ENSEMBL's v104 annotation. The data was obtained using  `EDASeq::getGeneLengthAndGCContent` function . 

Finally, the folder contains three PDF files with heatmap plots:
* **`TPM_Pcor_heatmap_HTSeq_counts_STAR2_7_9a_6352_rat_TCR_hepatocellular_carcinoma_RNAseq_samples.pdf`** Shows the heatmap with the hierachichal clustering of the Pearson correlation of all the possible paired compariosn across all the samples in the study using all the genes used for the counting.
* **`TPM_Pcor_heatmap_HTSeq_counts_STAR2_7_9a_6352_rat_TCR_hepatocellular_carcinoma_RNAseq_samples_Exp_group_annotation.pdf`** Same as above but with a colour anotation of the experimental group that the samples belong to and only based on proteing coding genes.
*  **`TPM_Pcor_heatmap_HTSeq_counts_STAR2_7_9a_6352_rat_TCR_hepatocellular_carcinoma_RNAseq_samples_Exp_group_annotation_NORIN0_samples.pdf`** Same as above but without the RIN 0 samples. 
* **`TPM_Pcor_heatmap_HTSeq_counts_STAR2_7_9a6352_rat_TCR_hepatocellular_carcinoma_RNAseq_ENSv104_samples_full_metadata.pdf`** Same as above but with additional annotations about the RIN quality of the sample of origin. 









