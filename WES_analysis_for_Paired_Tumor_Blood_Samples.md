WES_analysis_for_Paired_Tumor_Blood_Samples
================
Lian Chee Foong
2025-01-16

\#Step-by-Step Guide to Process Whole Exome Sequencing (WES) Data This
guide is tailored for users analyzing tumor and matched normal (blood)
WES datasets starting from clean FASTQ files.

I use Docker for Windows, which is highly recommended for its efficiency
and convenience.

### Setting Up the Environment

#### 1.1 Install Docker for Windows

Download and install Docker Desktop from Docker’s official site. Ensure
that: - Docker is running.

Now, open Windows’s Terminal, and perform the following tasks:

#### 1.2 Download Prebuilt Docker Images

``` bash
docker pull broadinstitute/gatk:latest
docker pull biocontainers/fastqc:v0.11.9_cv8
docker pull quay.io/biocontainers/multiqc:1.11--pyhdfd78af_0
docker pull biocontainers/samtools:v1.15.1_cv1
docker pull biocontainers/bwa:v0.7.17_cv1
```

#### 1.3 Set Up Directory Structure

Organize your data on your local machine (e.g., E:\_Sample_R001):

``` plaintext
E:\WES_Sample_R001\
├── fastq\
├── reference\
├── bam\
├── mutect2\
├── vcf\
├── annovar\
```

### 2. Quality Control of FASTQ Files

#### 2.1 Run FastQC

Use the FastQC container to assess the quality of your FASTQ files:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data biocontainers/fastqc:v0.11.9_cv8 \
    fastqc -o /data/qc /data/fastq/*.fastq
```

#### 2.2 Summarize Results with MultiQC

Ensure your FASTQ files are now located in ~/fastq.

Compile QC results using MultiQC:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data quay.io/biocontainers/multiqc:1.11--pyhdfd78af_0 \
    multiqc -o /data/qc /data/qc
Check the generated multiqc_report.html for quality metrics.
```

### 3. Align Reads to the Reference Genome

#### 3.1 Download Reference Genome

Place the reference genome files in E:\_Sample_R001. You can download
the GRCh38 reference using a browser or Docker:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data ubuntu bash -c 

apt-get update && apt-get install -y wget 

wget -P /data/reference https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz

gunzip /data/reference/hg38.fa.gz
```

#### 3.2 Index the Reference Genome

Use BWA, SAMtools, and GATK to index the reference:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data biocontainers/bwa:v0.7.17_cv1 \
    bwa index /data/reference/hg38.fa

docker run --rm -v E:\WES_Sample_R001:/data biocontainers/samtools:v1.15.1_cv1 \
    samtools faidx /data/reference/hg38.fa

docker run --rm -v E:\WES_Sample_R001:/data broadinstitute/gatk:latest \
    gatk CreateSequenceDictionary -R /data/reference/hg38.fa
```

#### 3.3 Align Reads

Align tumor and normal (paired blood) reads, respectively, using BWA:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data biocontainers/bwa:v0.7.17_cv1 \
    bwa mem -t 4 /data/reference/hg38.fa /data/fastq/tumor_R1.fastq /data/fastq/tumor_R2.fastq > /data/bam/tumor.sam

docker run --rm -v E:\WES_Sample_R001:/data biocontainers/bwa:v0.7.17_cv1 \
    bwa mem -t 4 /data/reference/hg38.fa /data/fastq/normal_R1.fastq /data/fastq/normal_R2.fastq > /data/bam/normal.sam
```

#### 3.4 Convert SAM to BAM and Sort

Convert and sort the SAM files using GATK:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data broadinstitute/gatk:latest \
    gatk --java-options "-Xmx8G" SortSam -I /data/bam/tumor.sam -O /data/bam/tumor_sorted.bam -SO coordinate

docker run --rm -v E:\WES_Sample_R001:/data broadinstitute/gatk:latest \
    gatk --java-options "-Xmx8G" SortSam -I /data/bam/normal.sam -O /data/bam/normal_sorted.bam -SO coordinate
```

Check the BAM headers to ensure proper sorting:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data biocontainers/samtools:v1.15.1_cv1 \
    samtools view -H /data/bam/tumor_sorted.bam | grep '@HD'
```

\*\*Make sure the header “HD” is “SO:coordinate”

### 4. Mark Duplicates and Base Quality Score Recalibration (BQSR)

#### 4.1 Mark Duplicates

Mark duplicates using GATK:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data broadinstitute/gatk:latest \
    gatk MarkDuplicates -I /data/bam/tumor_sorted.bam -O /data/bam/tumor_dedup.bam -M /data/bam/tumor_metrics.txt

docker run --rm -v E:\WES_Sample_R001:/data broadinstitute/gatk:latest \
    gatk MarkDuplicates -I /data/bam/normal_sorted.bam -O /data/bam/normal_dedup.bam -M /data/bam/normal_metrics.txt
```

#### 4.2 Index Deduplicated BAM Files

``` bash
docker run --rm -v E:\WES_Sample_R001:/data biocontainers/samtools:v1.15.1_cv1 \
    samtools index /data/bam/tumor_dedup.bam

docker run --rm -v E:\WES_Sample_R001:/data biocontainers/samtools:v1.15.1_cv1 \
    samtools index /data/bam/normal_dedup.bam
```

#### 4.3 Perform BQSR

Download known sites (e.g., dbSNP):

``` bash
wget -P E:\WES_Sample_R001\reference https://ftp.ncbi.nih.gov/snp/organisms/human_9606/VCF/00-common_all.vcf.gz
```

Run BQSR:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data broadinstitute/gatk:latest \
    gatk BaseRecalibrator -R /data/reference/hg38.fa -I /data/bam/tumor_dedup.bam \
    --known-sites /data/reference/00-common_all.vcf.gz -O /data/bam/tumor_recal.table

docker run --rm -v E:\WES_Sample_R001:/data broadinstitute/gatk:latest \
    gatk ApplyBQSR -R /data/reference/hg38.fa -I /data/bam/tumor_dedup.bam \
    --bqsr-recal-file /data/bam/tumor_recal.table -O /data/bam/tumor_bqsr.bam
```

### 5. Variant Calling with Mutect2

#### 5.1 Run Mutect2

Download required resources (germline, panel of normals) from GATK
Resource Hub.

From GATK Resource Hub
<https://console.cloud.google.com/storage/browser/gatk-best-practices/somatic-hg38;tab=objects?pli=1&prefix=&forceOnObjectsSortingFiltering=false>
– Germline Resource: af-only-gnomad.hg38.vcf.gz &
af-only-gnomad.hg38.vcf.gz.tbi – Panel of normal: 1000g_pon.hg38.vcf.gz
& 1000g_pon.hg38.vcf.gz.tbi

Run Mutect2:

``` bash
docker run --rm -v E:\WES_Sample_R001:/data broadinstitute/gatk:latest \
    gatk Mutect2 -R /data/reference/hg38.fa -I /data/bam/tumor_bqsr.bam -I /data/bam/normal_bqsr.bam \
    --germline-resource /data/reference/af-only-gnomad.hg38.vcf.gz \
    --panel-of-normals /data/reference/1000g_pon.hg38.vcf.gz \
    --f1r2-tar-gz /data/mutect2/f1r2.tar.gz \
    -O /data/vcf/mutect2.vcf.gz
```

\*\*This step takes relatively long time for a home desktop with 128 GB
RAM (Elapsed time: ~250-350 minutes)

Collect pileup summaries for both tumor and normal samples

``` bash
gatk GetPileupSummaries \
--java-options '-Xmx50G' --tmp-dir /data/mutect2/tmp/ \
-I /data/Bam/tumor_bqsr.bam \
-V /data/GATK_Resource_Hub/af-only-gnomad.hg38.vcf.gz \
-L /data/mutect2/exome_calling_regions.v1.1.interval_list \
-O /data/mutect2/Tumor_getpileupsummaries.table

gatk GetPileupSummaries \
--java-options '-Xmx50G' --tmp-dir /data/mutect2/tmp/ \
-I /data/Bam/normal_bqsr.bam \
-V /data/GATK_Resource_Hub/af-only-gnomad.hg38.vcf.gz \
-L /data/mutect2/exome_calling_regions.v1.1.interval_list \
-O /data/mutect2/Normal_getpileupsummaries.table
```

Use the pileup summaries to calculate contamination

``` bash
gatk CalculateContamination \
  -I /data/mutect2/Tumor_getpileupsummaries.table \
  --matched /data/mutect2/Normal_getpileupsummaries.table \
  -O /data/mutect2/pair_calculatecontamination.table
Learn Read Orientation Model

gatk LearnReadOrientationModel \
  -I /data/mutect2/BRCA001_f1r2.tar.gz \
  -O /data/mutect2/read-orientation-model.tar.gz
```

#### 5.2 Filter Mutect2 Calls

Apply filtering based on contamination and orientation model:

``` bash
gatk FilterMutectCalls -V /data/vcf/mutect2.vcf.gz -R /data/reference/hg38.fa 
--contamination-table /data/mutect2/pair_calculatecontamination.table 
--ob-priors /data/mutect2/read-orientation-model.tar.gz 
-O  /data/vcf/filtered.vcf.gz
```

### 6. Annotate Variants

Use ANNOVAR or Funcotator to annotate filtered variants.

Firstly, download the required database (Referred to
<https://gist.github.com/fo40225/f135b50b3e47d0997098264c3d28e590> )

``` bash
docker run -it --rm -v E:\WES_liver_private:/data ubuntu bash

perl annotate_variation.pl --downdb --webfrom annovar --buildver hg38 avsnp151 humandb/   

perl annotate_variation.pl -buildver hg38 -downdb -webfrom annovar refGene humandb/

## Optional
cd /humandb && wget -c http://hgdownload.cse.ucsc.edu/goldenpath/hg38/database/cytoBand.txt.gz && gunzip -d cytoBand.txt.gz && mv cytoBand.txt hg38_cytoBand.txt

## Optional
perl annotate_variation.pl -buildver hg38 -downdb -webfrom annovar clinvar_ 20190305 humandb/
Use ANNOVAR for further annotation:

docker run -it --rm -v E:\WES_liver_private:/data ubuntu bash
```

``` bash
cd /data/annovar/annovar/

apt-get update && apt-get install -y wget

apt-get update && apt-get install -y perl && cpan install Pod::Usage 

perl convert2annovar.pl -format vcf4old /data/mutect2/filtered.vcf > /data/mutect2/variants.avinput

perl table_annovar.pl /data/mutect2/variants.avinput humandb/ -buildver hg38     -out variants.annotated     -protocol refGene,gnomad40_genome,avsnp151     -operation g,f,f     -nastring  .     -polish 
```

## Troubleshooting High Mutation Counts

If you observe an unusually high number of mutated sites (e.g., 244 in a
single gene), consider the following:

1.  Review Filtering Parameters: Ensure that the parameters used in
    FilterMutectCalls are appropriate for your dataset.
2.  Check Input Quality: Verify that your input BAM files are
    well-prepared, properly marked, and have been aligned correctly.
3.  Inspect Pileup Summaries: Analyze the pileup summaries for any
    discrepancies or unexpected patterns in coverage.
4.  Use Annovar to get the filtered outputs by adding protocol of
    gnomad40\_\_genome and avsnp151.

By following these steps carefully, you should achieve a more reliable
set of filtered variants suitable for further analysis and visualization
using tools like maftools.

### References

1.  HBC Training
    <https://github.com/hbctraining/variant_analysis/blob/main/lessons>
2.  GATK Resource
    <https://github.com/broadinstitute/gatk/tree/master/docs/mutect>
