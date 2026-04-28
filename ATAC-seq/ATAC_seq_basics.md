# ATAC-seq Data Analysis Pipeline

## Step 1: Upload Files

- Turn on the VM
- Use terminal on your local computer to upload files to the VM :

```bash
gcloud compute scp \
  --scp-flag=-O \
  /path/to/local/files/*.fastq.gz \
  USERNAME@VM_NAME:~/
```

---

## Step 2: Organise Files in Folder

- Move fastq files into the correct folder :

```bash
mv *.fastq.gz /path/to/destination_folder/
```

---

## Step 3: Run FastQC

- Go into the folder with the fastq.gz files
- Run FastQC on all files :

```bash
nohup fastqc *.fastq.gz \
  -o /path/to/output/1.fastqc_results/ \
  > fastqc.out 2>&1 &
```

- `*.fastq.gz` = all files with the suffix fastq.gz will be taken as input
- `-o` = destination folder
- `2>&1 &` = redirects both stdout and stderr to fastqc.out and runs in background

---

## Step 4: Review the QC

- Use MultiQC to review the FastQC output :

```bash
pip install multiqc --user
multiqc --version
multiqc /path/to/1.fastqc_results/ -o /path/to/multiqc_report/
```

- Download the HTML files to your computer to open in browser :

```bash
# Step 1: Create a bucket in VM
gsutil mb -l REGION gs://YOUR_BUCKET_NAME/

# Step 2: Compress the folder in VM
tar czvf fastqc_results.tar.gz 1.fastqc_results/

# Step 3: Upload the tar folder to bucket
gsutil cp /path/to/fastqc_results.tar.gz gs://YOUR_BUCKET_NAME/

# Step 4: Create a service account and JSON key
# Go to GCP Console → IAM & Admin → Service Accounts → Create Service Account
# Give it a name, grant it the role: owner, create a JSON key
# This downloads a .json file to your computer

# Step 5: Upload the JSON key to your VM
scp ~/Downloads/your-key.json USERNAME@VM_IP:~/json_keys/
```

- If the scp does not work, use a bucket instead :

```bash
# Create a json_key bucket
gsutil mb -p YOUR_PROJECT gs://YOUR_JSON_KEY_BUCKET/

# Upload the json key to the bucket from local terminal
gsutil cp ~/Downloads/your-key.json gs://YOUR_JSON_KEY_BUCKET/

# Copy from the bucket to the json_files folder in VM
gsutil cp gs://YOUR_JSON_KEY_BUCKET/your-key.json ~/json_keys/

# Make a signed shareable URL (valid for 7 days)
gsutil signurl -d 7d ~/json_keys/your-key.json gs://YOUR_BUCKET_NAME/fastqc_results.tar.gz

# Copy the URL and paste it in the browser — the file will download
```

---

## Step 5: Trimming

- Put all the fastq files in one folder
- Create a shell script `trim_reads_script.sh` and make it executable :

```bash
nano trim_reads_script.sh
chmod +x trim_reads_script.sh
```

- Script contents :

```bash
#!/bin/bash

INPUT_DIR=~/path/to/raw_data
OUTPUT_DIR=~/path/to/2.Trimmed_files

mkdir -p "$OUTPUT_DIR"

for R1 in ${INPUT_DIR}/*_R1*.fastq.gz
do
  filename=$(basename "$R1")
  prefix=${filename%_R1*}
  R2="${INPUT_DIR}/${prefix}_R2_001.fastq.gz"
  OUT_R1="${OUTPUT_DIR}/${prefix}_R1.trimmed.fastq.gz"
  OUT_R2="${OUTPUT_DIR}/${prefix}_R2.trimmed.fastq.gz"

  echo "==== Trimming adapters for $prefix ===="

  fastp \
    -i "$R1" -I "$R2" \
    -o "$OUT_R1" -O "$OUT_R2" \
    --detect_adapter_for_pe \
    --thread 8 \
    --html "${OUTPUT_DIR}/${prefix}_fastp.html" \
    --json "${OUTPUT_DIR}/${prefix}_fastp.json"

  echo "==== Done trimming $prefix ===="
done
```

- Run it :

```bash
nohup ./trim_reads_script.sh > trim_log.txt 2>&1 &
```

---

## Step 6: Align the Reads Using Bowtie2

- Create a new folder for aligned reads :

```bash
mkdir ~/path/to/3.Aligned_reads
```

- Create the shell script `align_reads.sh` :

```bash
nano align_reads.sh
```

- Script contents :

```bash
#!/bin/bash

INDEX_PATH=~/path/to/Bowtie2_index/GRCh38_noalt_as/GRCh38_noalt_as
TRIMMED_DIR=~/path/to/2.Trimmed_files
OUTPUT_DIR=~/path/to/3.Aligned_reads
LOGFILE=alignment.log

mkdir -p $OUTPUT_DIR

echo "Starting Bowtie2 alignment pipeline..." | tee "$LOGFILE"

for R1 in "$TRIMMED_DIR"/*_R1.trimmed.fastq.gz; do
  BASENAME=$(basename "$R1" _R1.trimmed.fastq.gz)
  R2="$TRIMMED_DIR/${BASENAME}_R2.trimmed.fastq.gz"

  echo "Processing sample: $BASENAME" | tee -a "$LOGFILE"

  if [[ ! -f "$R2" ]]; then
    echo "Skipping $BASENAME: Missing $R2" | tee -a "$LOGFILE"
    continue
  fi

  bowtie2 -x "$INDEX_PATH" \
    -1 "$R1" -2 "$R2" \
    --no-mixed --no-discordant --no-unal \
    -p 8 2>>"$LOGFILE" | \
    samtools view -@ 4 -bS - | \
    samtools sort -@ 4 -o "$OUTPUT_DIR/${BASENAME}.sorted.bam" -

  samtools index "$OUTPUT_DIR/${BASENAME}.sorted.bam"

  echo "Done: $BASENAME" | tee -a "$LOGFILE"
done

echo "All alignments completed!" | tee -a "$LOGFILE"
```

- Run it :

```bash
nohup ~/path/to/align_reads.sh > ~/path/to/3.Aligned_reads/align_reads_run.log 2>&1 &
```

---

## Step 7: QC of Aligned Files

### Step 7a: Filter BAMs

```bash
#!/bin/bash

INPUT_DIR=~/path/to/3.Aligned_reads
OUTPUT_DIR=~/path/to/4.Filtered_bams

mkdir -p $OUTPUT_DIR

for BAM in $INPUT_DIR/*.sorted.bam; do
  BASENAME=$(basename "$BAM" .sorted.bam)
  echo "Filtering $BASENAME"
  samtools view -@ 4 -b -q 30 -F 1804 "$BAM" > "$OUTPUT_DIR/${BASENAME}.q30.bam"
  samtools index "$OUTPUT_DIR/${BASENAME}.q30.bam"
done
```

### Step 7b: Deduplication

```bash
#!/bin/bash

INPUT_DIR=~/path/to/4.Filtered_bams
OUTPUT_DIR=~/path/to/5.Deduplicated_bams

mkdir -p $OUTPUT_DIR

for BAM in $INPUT_DIR/*.q30.bam; do
  BASENAME=$(basename "$BAM" .q30.bam)
  echo "Deduplicating $BASENAME"

  samtools sort -n -@ 4 "$BAM" -o "$OUTPUT_DIR/${BASENAME}.qname.bam"
  samtools fixmate -m "$OUTPUT_DIR/${BASENAME}.qname.bam" "$OUTPUT_DIR/${BASENAME}.fixmate.bam"
  samtools sort -@ 4 "$OUTPUT_DIR/${BASENAME}.fixmate.bam" -o "$OUTPUT_DIR/${BASENAME}.sort.bam"
  samtools markdup -r "$OUTPUT_DIR/${BASENAME}.sort.bam" "$OUTPUT_DIR/${BASENAME}.rmdup.bam"
  samtools index "$OUTPUT_DIR/${BASENAME}.rmdup.bam"

  rm "$OUTPUT_DIR/${BASENAME}.qname.bam" "$OUTPUT_DIR/${BASENAME}.fixmate.bam" "$OUTPUT_DIR/${BASENAME}.sort.bam"
done
```

### Step 7c: Remove Mitochondrial Reads

```bash
#!/bin/bash

INPUT_DIR=~/path/to/5.Deduplicated_bams
OUTPUT_DIR=~/path/to/6.NoMT_bams

mkdir -p $OUTPUT_DIR

for BAM in $INPUT_DIR/*.rmdup.bam; do
  BASENAME=$(basename "$BAM" .rmdup.bam)
  echo "Removing mitochondrial reads for $BASENAME"

  samtools idxstats "$BAM" | cut -f 1 | grep -v 'chrM' | \
    xargs samtools view -b "$BAM" > "$OUTPUT_DIR/${BASENAME}.rmdup.noMT.bam"

  samtools index "$OUTPUT_DIR/${BASENAME}.rmdup.noMT.bam"
done
```

### Step 7d: Remove Blacklist Regions

```bash
#!/bin/bash

INPUT_DIR=~/path/to/6.NoMT_bams
OUTPUT_DIR=~/path/to/7.Final_filtered_bams
BLACKLIST=~/path/to/reference/hg38-blacklist.v2.bed

mkdir -p $OUTPUT_DIR

for BAM in $INPUT_DIR/*.rmdup.noMT.bam; do
  BASENAME=$(basename "$BAM" .rmdup.noMT.bam)
  echo "Removing blacklist regions for $BASENAME"

  if command -v bedtools &> /dev/null; then
    bedtools intersect -v -a "$BAM" -b $BLACKLIST > "$OUTPUT_DIR/${BASENAME}.filtered.bam"
    samtools index "$OUTPUT_DIR/${BASENAME}.filtered.bam"
  else
    echo "bedtools not found. Copying BAM without blacklist filtering"
    cp "$BAM" "$OUTPUT_DIR/${BASENAME}.filtered.bam"
    samtools index "$OUTPUT_DIR/${BASENAME}.filtered.bam"
  fi
done
```

---

## Step 8: Peak Calling and Metadata Creation

### Step 8a: Call Peaks with MACS2

```bash
#!/bin/bash

BAM_DIR=~/path/to/7.Final_filtered_bams
OUTDIR=~/path/to/8.Peaks
GENOME=hs  # hs = human, mm = mouse
THREADS=8

mkdir -p "$OUTDIR"

for BAM in $BAM_DIR/*.filtered.bam; do
  SAMPLE=$(basename "$BAM" .filtered.bam)
  echo "Calling peaks for $SAMPLE"

  macs2 callpeak \
    -t "$BAM" \
    -f BAMPE \
    -g $GENOME \
    -n $SAMPLE \
    --outdir "$OUTDIR" \
    --nomodel --shift -100 --extsize 200 -q 0.01

  echo "Done: $SAMPLE"
done

echo "All peak calling jobs finished."
```

### Step 8b: Create Union Consensus Peaks

```bash
#!/bin/bash

PEAK_DIR=~/path/to/8.Peaks
OUTFILE=~/path/to/9.consensus_peaks/consensus_union.bed

mkdir -p "$(dirname $OUTFILE)"

echo "Creating union consensus peak set from all samples..."

cat $PEAK_DIR/*_peaks.narrowPeak \
  | cut -f1-3 \
  | sort -k1,1 -k2,2n \
  | bedtools merge -i - > $OUTFILE

awk 'BEGIN{OFS="\t"}{print $1,$2,$3,"peak_union_"NR}' $OUTFILE > ${OUTFILE%.bed}_withID.bed

echo "Union consensus peaks written to:"
echo "  - $OUTFILE"
echo "  - ${OUTFILE%.bed}_withID.bed"
```

### Step 8c: Create Peak Count Matrix

```bash
#!/bin/bash

PEAKS=~/path/to/9.consensus_peaks/consensus_union_withID.bed
BAM_DIR=~/path/to/7.Final_filtered_bams
OUTFILE=~/path/to/9.consensus_peaks/peak_counts_matrix.tsv

BAMS=($BAM_DIR/*.filtered.bam)

echo "Counting reads for ${#BAMS[@]} BAM files"

bedtools multicov -bams "${BAMS[@]}" -bed "$PEAKS" > ${OUTFILE}.tmp

{
  echo -ne "chr\tstart\tend\tpeak_id"
  for bam in "${BAMS[@]}"; do
    sample=$(basename "$bam" .filtered.bam)
    echo -ne "\t$sample"
  done
  echo
  cat ${OUTFILE}.tmp
} > "$OUTFILE"

rm ${OUTFILE}.tmp

echo "Peak count matrix written to $OUTFILE"
```

### Step 8d: Create Sample Metadata File

```bash
#!/bin/bash

BAM_DIR=~/path/to/7.Final_filtered_bams
OUTFILE=~/path/to/9.consensus_peaks/sample_metadata.tsv

mkdir -p $(dirname "$OUTFILE")

BAMS=($BAM_DIR/*.filtered.bam)

echo -e "sample\tcondition\tbam_path" > "$OUTFILE"

for bam in "${BAMS[@]}"; do
  sample=$(basename "$bam" .filtered.bam)
  lname=$(echo "$sample" | tr '[:upper:]' '[:lower:]')

  if [[ "$lname" == *"pos"* ]]; then
    cond="pos"
  elif [[ "$lname" == *"neg"* ]]; then
    cond="neg"
  else
    cond="unknown"
  fi

  echo -e "${sample}\t${cond}\t${bam}" >> "$OUTFILE"
done

echo "Metadata file written to $OUTFILE"
echo "Please check and edit if any samples were marked 'unknown'"
```

---

## Downstream Analysis

From the peak counts / consensus union peak count matrix the following analyses can be performed :

- PCA and DESeq2 — differential accessibility analysis
- HOMER — motif enrichment analysis (see HOMER_analysis.md)
- chromVAR — transcription factor activity scoring
