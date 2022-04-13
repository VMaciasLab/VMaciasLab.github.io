#!/bin/bash -x

START=$(date +%s);

for sample in "SMU"

do

    java -Xmx32768M -jar /ngs-data/share_tools/tools/Trimmomatic-0.40/dist/jar/trimmomatic-0.40-rc1.jar PE -threads 30 -phred33 \
    ${sample}_R1.fastq.gz ${sample}_R2.fastq.gz \
    ./trimmed/$sample'_1_paired.fastq.gz' ./trimmed/$sample'_1_unpaired.fastq.gz' ./trimmed/$sample'_2_paired.fastq.gz' ./trimmed/$sample'_2_unpaired.fastq.gz' \
    ILLUMINACLIP:/home/alfano/bin/Trimmomatic-0.39/adapters/TruSeq3-PE.fa:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36

	bwa mem -t 16 \
	/ngs-data/share_ref/bundle/current/Aedes_albopictus/AlbCanu/GCF_006496715.1_Aalbo_primary.1_genomic_clean.fasta \
	./trimmed/$sample'_1_paired.fastq.gz' \
	./trimmed/$sample'_2_paired.fastq.gz' \
	-R "@RG\tID:${sample}\tLB:WholeGenome\tSM:${sample}\tPL:ILLUMINA\tPU:FlowCellId" | \
	samtools view -@ 16 -b - > ./alignments/${sample}.bam

    rm ./trimmed/$sample'_1_unpaired.fastq.gz' ./trimmed/$sample'_2_unpaired.fastq.gz'

	java -Xmx64768M -jar /ngs-data/share_tools/tools/picard_2.26.10/picard.jar CleanSam \
	INPUT= ./alignments/${sample}.bam \
	OUTPUT= ./alignments/${sample}_cleaned.bam \
	VALIDATION_STRINGENCY=SILENT

	java -Xmx64768M -jar /ngs-data/share_tools/tools/picard_2.26.10/picard.jar SortSam \
	INPUT= ./alignments/${sample}_cleaned.bam \
	OUTPUT= ./alignments/${sample}_sorted.bam \
	SORT_ORDER=coordinate \
	CREATE_INDEX=True

	java -Xmx64768M -jar /ngs-data/share_tools/tools/picard_2.26.10/picard.jar MarkDuplicates \
	INPUT= ./alignments/${sample}_sorted.bam \
	OUTPUT= ./markdup/${sample}_markdup.bam \
	METRICS_FILE= ./markdup/${sample}_markdup_metrics.txt \
	ASSUME_SORTED=True \
	CREATE_INDEX=True

	samtools index -@ 16 ./markdup/${sample}_markdup.bam

	java -Xmx32768M -jar /ngs-data/share_tools/tools/picard_2.26.10/picard.jar ValidateSamFile \
	INPUT= ./markdup/${sample}_markdup.bam \
	O= ./markdup/$sample'_markdup_ValidateSamFile.txt' \
	MODE=SUMMARY

done