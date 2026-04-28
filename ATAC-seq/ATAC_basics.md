ATAC SEQ data analysis

[Step 1: Upload files]{.mark}

1.  Turn on the VM

2.  Use terminal in the local computer and :

> Code:
>
> gcloud compute scp \\
>
> \--scp-flag=-O \\
>
> /Users/tat/Documents/scRNAseq_2024/Snoeck_lab/Juan/ATAC_SEQ/ATAC_seq_2/\*.fastq.gz
> \\ \## this is the file path
>
> taniathimraj1@atac-seq-vm:\~/ \\ \## this is the username and VM name

[Step 2 : Organize the files in the folder]{.mark}

1.  mv \*.fastq.gz destination_folder

[Step 3 : Run fastqc]{.mark}

1.  go into the folder with the fastq.gz files

2.  from there :

Code:

> nohup fastqc \*.fastq.gz \\ ##all files with the suffix fastq.gz will
> be taken as input
>
> -o \~/ATAC2/atac2_analysis/1.fastqc_results \\ \## destination folder
>
> \> fastqc.out 2\>&1 & \## redirects both standard output and standard
> error to fastqc.out (instead of nohup.out) "&" explicitly puts the
> process in the background immediately.

[Step 4: Review the QC]{.mark}

1.  Use multiqc to review the fastqc output

> Code:
>
> pip install multiqc \--user ##so no need for admin permissions etc.,
>
> multiqc \--version ##first check if its installed and if it's not
> installed :

multiqc \~/ATAC2/atac2_analysis/1.fastqc_results/ \\

> -o \~/ATAC2/atac2_analysis/multiqc_report

2.  Download the html files to the computer to open in browser :

Step 1: Create a bucket in VM

> gsutil mb -l asia-south1 gs://my-atac-fastqc/
>
> Step 2: compress the folder in VM
>
> tar czvf fastqc_results.tar.gz 1.fastqc_results/
>
> Step 3: upload the tar folder in VM
>
> gsutil cp \~/ATAC2/atac2_analysis/fastqc_results.tar.gz
> gs://my-atac-fastqc/
>
> Step 4: Create a service account and JSON key in VM
>
> Go to GCP Console → IAM & Admin → Service Accounts → Create Service
> Account.
>
> Give it a name (e.g., fastqc).
>
> Grant it the role: owner.
>
> Create a JSON key. This downloads a .json file to your computer.
>
> Step 5:Upload the JSON key to your VM from terminal
>
> This should generally work
>
> Scp \\
>
> \~/Downloads/juan-atacseq-analysis-abcbb6ee94a5.json\\
> \[USER\]@\[VM_IP\]:\~/json_keys/

IF IT DOES NOT WORK:

> Create a json_key bucket on VM :
>
> gsutil mb -p juan-atacseq-analysis gs://juan-json-key-bucket-2025/
>
> Upload the json_key to the bucket in terminal:
>
> gsutil cp /Users/tat/Downloads/juan-atacseq-analysis-abcbb6ee94a5.json
> gs://juan-json-key-bucket-2025/
>
> Copy from the bucket to the json_files folder in terminal:
>
> gsutil cp
> gs://juan-json-key-bucket-2025/juan-atacseq-analysis-abcbb6ee94a5.json
> \~/json_keys/
>
> make a signed sharable URL:
>
> gsutil signurl -d 7d
> \~/json_keys/juan-atacseq-analysis-abcbb6ee94a5.json
> gs://my-atac-fastqc/fastqc_results.tar.gz
>
> copy the url and paste it in the browser and the file will download

[Step 5: Trimming]{.mark}

1.  Put all the fastq files in one folder (fastq_rawdata)

2.  Create a shell script: trim_reads_script.sh and make it executable
    by chmod +x scipt.sh

> Code :
>
> nano trim_reads_script.sh
>
> #!/bin/bash
>
> \# Input and output directories
>
> INPUT_DIR=\~/ATAC2/atac2_raw_data
>
> OUTPUT_DIR=\~/ATAC2/atac2_analysis/2.Trimmed_files
>
> mkdir -p \"\$OUTPUT_DIR\"
>
> \# Automatically find all R1 files
>
> for R1 in \${INPUT_DIR}/\*\_R1\*.fastq.gz
>
> do
>
> \# Derive the prefix by removing the \_R1_001.fastq.gz part
>
> filename=\$(basename \"\$R1\")
>
> prefix=\${filename%\_R1\*}
>
> \# Define the corresponding R2 file
>
> R2=\"\${INPUT_DIR}/\${prefix}\_R2_001.fastq.gz\"
>
> \# Output files
>
> OUT_R1=\"\${OUTPUT_DIR}/\${prefix}\_R1.trimmed.fastq.gz\"
>
> OUT_R2=\"\${OUTPUT_DIR}/\${prefix}\_R2.trimmed.fastq.gz\"
>
> echo \"==== Trimming adapters for \$prefix ====\"
>
> \# Run fastp
>
> fastp \\
>
> -i \"\$R1\" -I \"\$R2\" \\
>
> -o \"\$OUT_R1\" -O \"\$OUT_R2\" \\
>
> \--detect_adapter_for_pe \\
>
> \--thread 8 \\
>
> \--html \"\${OUTPUT_DIR}/\${prefix}\_fastp.html\" \\
>
> \--json \"\${OUTPUT_DIR}/\${prefix}\_fastp.json\"
>
> echo \"==== Done trimming \$prefix ====\"
>
> done
>
> \^c enter \^x
>
> chmod +x trim_reads_script.sh

3.  Run it

Code :

nohup ./trim_reads_script.sh \> trim_log.txt 2\>&1 &

[Step 6: Align the reads using Bowtie2]{.mark}

1.  Create a new folder 3.Aligned_reads

Code:

mkdir \~/ATAC2/atac2_analysis/3.Aligned_reads

2.  create the shell script and save as align_reads.sh as follows :

code:

[nano align_reads.sh]{.mark}

\# Directory paths

INDEX_PATH=\~/Bowtie2_index/GRCh38_noalt_as/GRCh38_noalt_as

TRIMMED_DIR=\~/ATAC2/atac2_analysis/2.Trimmed_files

OUTPUT_DIR=\~/ATAC2/atac2_analysis/3.Aligned_reads

LOGFILE=alignment.log

\# Create output directory if it doesn\'t exist

mkdir -p \$OUTPUT_DIR

echo \" \^=\^z\^\` Starting Bowtie2 alignment pipeline\...\" \| tee
\"\$LOGFILE\"

\# ==== LOOP OVER SAMPLES ====

for R1 in \"\$TRIMMED_DIR\"/\*\_R1.trimmed.fastq.gz; do

BASENAME=\$(basename \"\$R1\" \_R1.trimmed.fastq.gz)

R2=\"\$TRIMMED_DIR/\${BASENAME}\_R2.trimmed.fastq.gz\"

echo \" \^=\^t\^d Processing sample: \$BASENAME\" \| tee -a
\"\$LOGFILE\"

\# Check if R2 exists

if \[\[ ! -f \"\$R2\" \]\]; then

echo \" \^z \^o Skipping \$BASENAME: Missing \$R2\" \| tee -a
\"\$LOGFILE\"

continue

fi

\# Run alignment + BAM conversion + sorting

bowtie2 -x \"\$INDEX_PATH\" \\

-1 \"\$R1\" -2 \"\$R2\" \\

\--no-mixed \--no-discordant \--no-unal \\

-p 8 2\>\>\"\$LOGFILE\" \| \\

samtools view -@ 4 -bS - \| \\

samtools sort -@ 4 -o \"\$OUTPUT_DIR/\${BASENAME}.sorted.bam\" -

\# Index BAM

samtools index \"\$OUTPUT_DIR/\${BASENAME}.sorted.bam\"

echo \" \^\|\^e Done: \$BASENAME\" \| tee -a \"\$LOGFILE\"

done

echo \" \^=\^n All alignments completed!\" \| tee -a \"\$LOGFILE\"

> \^O enter \^X

nohup \~/sh_files/align_reads.sh \> \\
\~/ATAC2/atac2_analysis/3.Aligned_reads/align_reads_run.log 2\>&1 &

[Step 7: QC of aligned files]{.mark}

Step 1: QC/Filtering the aligned bam files: ["4.filter_bams.sh"]{.mark}

[Code:]{.mark}

OUTPUT_DIR=\~/ATAC2/atac2_analysis/4.Filtered_bams

mkdir -p \$OUTPUT_DIR

for BAM in \$INPUT_DIR/\*.sorted.bam; do

BASENAME=\$(basename \"\$BAM\" .sorted.bam)

echo \"Filtering \$BASENAME\"

samtools view -@ 4 -b -q 30 -F 1804 \"\$BAM\" \>
\"\$OUTPUT_DIR/\${BASENAME}.q30.bam\"

samtools index \"\$OUTPUT_DIR/\${BASENAME}.q30.bam\"

done

Step 2: deduplication of the filtered files [: 4.dedup.sh]{.mark}

> INPUT_DIR=\~/ATAC2/atac2_analysis/4.Filtered_bams
>
> OUTPUT_DIR=\~/ATAC2/atac2_analysis/5.Deduplicated_bams
>
> mkdir -p \$OUTPUT_DIR
>
> for BAM in \$INPUT_DIR/\*.q30.bam; do
>
> BASENAME=\$(basename \"\$BAM\" .q30.bam)
>
> echo \"Deduplicating \$BASENAME\"
>
> #Sort by queryname
>
> samtools sort -n -@ 4 \"\$BAM\" -o
> \"\$OUTPUT_DIR/\${BASENAME}.qname.bam\"
>
> \# Fixmate
>
> samtools fixmate -m \"\$OUTPUT_DIR/\${BASENAME}.qname.bam\"
> \"\$OUTPUT_DIR/\${BASENAME}.fixmate.bam\"
>
> \# Sort by coordinates (required for markdup)
>
> samtools sort -@ 4 \"\$OUTPUT_DIR/\${BASENAME}.fixmate.bam\" -o
> \"\$OUTPUT_DIR/\${BASENAME}.sort.bam\"
>
> \# Mark duplicates
>
> samtools markdup -r \"\$OUTPUT_DIR/\${BASENAME}.sort.bam\"
> \"\$OUTPUT_DIR/\${BASENAME}.rmdup.bam\"
>
> \# Index final BAM
>
> samtools index \"\$OUTPUT_DIR/\${BASENAME}.rmdup.bam\"
>
> \# Clean up intermediate files
>
> rm \"\$OUTPUT_DIR/\${BASENAME}.qname.bam\"
> \"\$OUTPUT_DIR/\${BASENAME}.fixmate.bam\"
> \"\$OUTPUT_DIR/\${BASENAME}.sort.bam\"
>
> done
>
> Step 3: remove MT: [5.remove_MT.sh]{.mark}
>
> #!/bin/bash
>
> INPUT_DIR=\~/ATAC2/atac2_analysis/5.Deduplicated_bams
>
> OUTPUT_DIR=\~/ATAC2/atac2_analysis/6.NoMT_bams
>
> mkdir -p \$OUTPUT_DIR
>
> for BAM in \$INPUT_DIR/\*.rmdup.bam; do
>
> BASENAME=\$(basename \"\$BAM\" .rmdup.bam)
>
> echo \"Removing mitochondrial reads for \$BASENAME\"
>
> samtools idxstats \"\$BAM\" \| cut -f 1 \| grep -v \'chrM\' \| \\
>
> xargs samtools view -b \"\$BAM\" \>
> \"\$OUTPUT_DIR/\${BASENAME}.rmdup.noMT.bam\"
>
> samtools index \"\$OUTPUT_DIR/\${BASENAME}.rmdup.noMT.bam\"
>
> done
>
> Step 4: remove blacklist : [6.blacklist_removal.sh]{.mark}
>
> #!/bin/bash
>
> INPUT_DIR=\~/ATAC2/atac2_analysis/6.NoMT_bams
>
> OUTPUT_DIR=\~/ATAC2/atac2_analysis/7.Final_filtered_bams
>
> BLACKLIST=\~/reference/hg38-blacklist.v2.bed
>
> mkdir -p \$OUTPUT_DIR
>
> for BAM in \$INPUT_DIR/\*.rmdup.noMT.bam; do
>
> BASENAME=\$(basename \"\$BAM\" .rmdup.noMT.bam)
>
> echo \"Removing blacklist regions for \$BASENAME\"
>
> if command -v bedtools &\> /dev/null; then
>
> bedtools intersect -v -a \"\$BAM\" -b \$BLACKLIST \>
> \"\$OUTPUT_DIR/\${BASENAME}.filtered.bam\"
>
> samtools index \"\$OUTPUT_DIR/\${BASENAME}.filtered.bam\"
>
> else
>
> echo \" \^z \^o bedtools not found. Copying BAM without blacklist
> filtering\"
>
> cp \"\$BAM\" \"\$OUTPUT_DIR/\${BASENAME}.filtered.bam\"
>
> samtools index \"\$OUTPUT_DIR/\${BASENAME}.filtered.bam\"
>
> fi
>
> done

Step 8 Peak calling and metadata creation []{.mark}

[7. call_peaks.sh]{.mark}

> \# Description: Call ATAC-seq peaks with MACS2 for all filtered BAMs.
>
> \# ==== CONFIG ====
>
> BAM_DIR=\~/ATAC2/atac2_analysis/7.Final_filtered_bams \# folder with
> final filtered BAMs
>
> OUTDIR=\~/ATAC2/atac2_analysis/8.Peaks \# output folder for peaks
>
> GENOME=hs \# genome size: hs (human), mm (mouse)
>
> THREADS=8 \# adjust based on VM
>
> mkdir -p \"\$OUTDIR\"
>
> \# ==== LOOP ====
>
> for BAM in \$BAM_DIR/\*.filtered.bam; do
>
> SAMPLE=\$(basename \"\$BAM\" .filtered.bam)
>
> echo \" \^=\^t\^d Calling peaks for \$SAMPLE\"
>
> macs2 callpeak \\
>
> -t \"\$BAM\" \\
>
> -f BAMPE \\
>
> -g \$GENOME \\
>
> -n \$SAMPLE \\
>
> \--outdir \"\$OUTDIR\" \\
>
> \--nomodel \--shift -100 \--extsize 200 -q 0.01
>
> echo \" \^\|\^e Done: \$SAMPLE\"
>
> done
>
> echo \" \^=\^n All peak calling jobs finished.\"

[8.union_consensus_peaks.sh]{.mark}

PEAK_DIR=\~/ATAC2/atac2_analysis/8.Peaks

OUTFILE=\~/ATAC2/atac2_analysis/9.consensus_peaks/consensus_union.bed

mkdir -p \"\$PEAK_DIR\"

echo \" \^=\^z\^\` Creating union consensus peak set from all
samples\...\"

\# Merge all narrowPeak files -\> chr start end only -\> sort -\> merge
overlaps

cat \$PEAK_DIR/\*\_peaks.narrowPeak \\

\| cut -f1-3 \\

\| sort -k1,1 -k2,2n \\

\| bedtools merge -i - \> \$OUTFILE

\# Add unique peak IDs

> awk \'BEGIN{OFS=\"\\t\"}{print \$1,\$2,\$3,\"peak_union\_\"NR}\'
> \$OUTFILE \> \${OUTFILE%.bed}\_withID.bed

echo \" \^\|\^e Union consensus peaks written to:\"

echo \" - \$OUTFILE\"

echo \" - \${OUTFILE%.bed}\_withID.bed\"

[9.peak_count_matrix.sh]{.mark}

> PEAKS=\~/ATAC2/atac2_analysis/9.consensus_peaks/consensus_union_withID.bed
>
> BAM_DIR=\~/ATAC2/atac2_analysis/7.Final_filtered_bams
>
> OUTFILE=\~/ATAC2/atac2_analysis/9.consensus_peaks/peak_counts_matrix.tsv
>
> \# ==== CHECK INPUT ====
>
> if \[ ! -f \"\$PEAKS\" \]; then
>
> echo \" \^}\^l Consensus peaks file not found: \$PEAKS\"
>
> exit 1
>
> fi
>
> BAMS=(\$BAM_DIR/\*.filtered.bam)
>
> if \[ \${#BAMS\[@\]} -eq 0 \]; then
>
> echo \" \^}\^l No BAMs found in \$BAM_DIR\"
>
> exit 1
>
> fi
>
> echo \" \^=\^z\^\` Counting reads for \${#BAMS\[@\]} BAM files\"
>
> \# ==== RUN MULTICOV ====
>
> \# bedtools multicov will output: chr start end peak_id counts\...
>
> bedtools multicov -bams \"\${BAMS\[@\]}\" -bed \"\$PEAKS\" \>
> \${OUTFILE}.tmp
>
> \# ==== ADD HEADER ====
>
> {
>
> \# print header
>
> echo -ne \"chr\\tstart\\tend\\tpeak_id\"
>
> for bam in \"\${BAMS\[@\]}\"; do
>
> sample=\$(basename \"\$bam\" .filtered.bam)
>
> echo -ne \"\\t\$sample\"
>
> done
>
> echo
>
> \# append counts
>
> cat \${OUTFILE}.tmp
>
> } \> \"\$OUTFILE\"
>
> rm \${OUTFILE}.tmp
>
> echo \" \^\|\^e Peak count matrix written to \$OUTFILE\"

[10.peakcounts_metadata.sh]{.mark}

BAM_DIR=\~/ATAC2/atac2_analysis/7.Final_filtered_bams

OUTFILE=\~/ATAC2/atac2_analysis/9.consensus_peaks/sample_metadata.tsv

mkdir -p \$(dirname \"\$OUTFILE\")

\# ==== LIST BAM FILES ====

BAMS=(\$BAM_DIR/\*.filtered.bam)

if \[ \${#BAMS\[@\]} -eq 0 \]; then

echo \" \^}\^l No BAMs found in \$BAM_DIR\"

exit 1

fi

\# ==== WRITE HEADER ====

echo -e \"sample\\tcondition\\tbam_path\" \> \"\$OUTFILE\"

\# ==== ASSIGN CONDITIONS ====

for bam in \"\${BAMS\[@\]}\"; do

sample=\$(basename \"\$bam\" .filtered.bam)

lname=\$(echo \"\$sample\" \| tr \'\[:upper:\]\' \'\[:lower:\]\')

if \[\[ \"\$lname\" == \*\"pos\"\* \]\]; then

cond=\"pos\"

elif \[\[ \"\$lname\" == \*\"neg\"\* \]\]; then

cond=\"neg\"

else

cond=\"unknown\"

fi

echo -e \"\${sample}\\t\${cond}\\t\${bam}\" \>\> \"\$OUTFILE\"

done

echo \" \^\|\^e Metadata file written to \$OUTFILE\"

echo \" \^=\^q\^i Please check and edit if any samples were marked
\'unknown\'\"

> DOWNSTREAM ANALYSIS

From the peak counts / consensus union peak count matrix we can do the
following :

1.  PCA and DESEQ2

2.  HOMER

3.  chromVAR

<!-- -->

1.  PCA and DESEQ2:
