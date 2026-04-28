# HOMER Motif Enrichment Analysis

## What is HOMER?

HOMER (Hypergeometric Optimization of Motif EnRichment) is a toolkit for analyzing high-throughput sequencing data, particularly motif enrichment in genomic regions.

- You give HOMER a set of genomic regions (e.g. ATAC-seq peaks that gain or lose accessibility)
- It looks inside those regions for recurring DNA sequence patterns (motifs)
- It compares these patterns to known transcription factor (TF) motifs in its database
- It outputs which transcription factors might be binding in your regions of interest

**Why it's useful for ATAC-seq :** ATAC-seq identifies open chromatin regions which correspond to active regulatory elements. Motif enrichment tells you which TFs might be driving the accessibility changes you observe.

---

## Installation

- Download and install HOMER :
  - `wget http://homer.ucsd.edu/homer/configureHomer.pl`
  - `perl configureHomer.pl -install`
- Install genome (e.g. hg38) : `perl ~/homer/configureHomer.pl -install hg38`
- Update HOMER : `perl ~/homer/configureHomer.pl -update`
- Verify installation : `findMotifsGenome.pl --help`

---

## Input Files

HOMER requires BED format files. A BED file looks like this:

```
chr1    1000    2000    peak1    .    .
chr1    5000    6000    peak2    .    .
```

Columns: `chromosome`, `start`, `end`, `name`, `score`, `strand`

**Creating BED files from DESeq2 output :**

- HOMER analysis begins with DESeq2 output from ATAC-seq differential accessibility analysis
- Significant UP = increased accessibility in condition of interest → query/positive file
- Significant DOWN = decreased accessibility → background/negative file
- Filter : padj < 0.05

---

## Running HOMER — Known Motif Analysis

- Basic command :
  - `findMotifsGenome.pl query.bed hg38 /path/to/output/ -bg background.bed`
- With size parameter (200bp is standard) :
  - `findMotifsGenome.pl query.bed hg38 /path/to/output/ -bg background.bed -size 200`
- Parameters :
  - `query.bed` = your regions of interest (e.g. upregulated peaks)
  - `hg38` = reference genome
  - `/path/to/output/` = where results will be saved
  - `-bg background.bed` = background regions (e.g. downregulated peaks)
  - `-size 200` = window size around peak center to search for motifs

---

## Output Files

- `knownResults.html` — visual summary of known motif enrichment
- `knownResults.txt` — table of known motifs with statistics
- `homerResults.html` — visual summary of de novo motifs
- `homerMotifs.all.motifs` — all de novo motifs found
- `homerResults/motif*.motif` — individual de novo motif files

**How to interpret knownResults.txt :** Filter for q-value (Benjamini) < 0.05 AND % Target > % Background

---

## Known Motifs Analysis — Full Workflow

- Step 1 : Identify enriched TFs from knownResults.txt — filter for q-value < 0.05 and % target > % background (done manually in Excel)
- Step 2 : Run annotatePeaks.pl for each motif to find peaks with TF binding sites :
  - `annotatePeaks.pl query.bed hg38 -m /path/to/TF.motif > /path/to/output/TF_peaks_with_motif.txt`
- Step 3 : Annotate all peaks to genes :
  - `annotatePeaks.pl query.bed hg38 > /path/to/output/annotated_peaks.txt`
- Step 4 : Fix missing genome annotation if needed :
  - `perl ~/homer/configureHomer.pl -install hg38`
- Step 5 : Check gene names are populated :
  - `awk -F'\t' 'NR>1 && $17!="" {print $17}' annotated_peaks.txt | head -20`
- Step 6 : Extract peak IDs with motif hits :
  - `awk -F'\t' 'NR>1 && $22!="" {print $1}' TF_peaks_with_motif.txt > TF_motif_peak_ids.txt`
- Step 7 : Link motif peaks to genes :
  - `awk -F'\t' 'NR==1{next} FNR==NR{ids[$1]=1; next} ids[$1] && $17!="" {print $1"\t"$17"\t"$10"\t"$8}' TF_motif_peak_ids.txt annotated_peaks.txt > TF_target_genes.txt`
- Step 8 : Automate for multiple TFs with a bash script :

```bash
#!/bin/bash
for motif in /path/to/motifs/*.motif; do
    tf=$(basename $motif .motif)
    echo "Processing $tf..."
    annotatePeaks.pl query.bed hg38 -m $motif > /path/to/output/${tf}_peaks_with_motif.txt
    awk -F'\t' 'NR>1 && $22!="" {print $1}' /path/to/output/${tf}_peaks_with_motif.txt > /path/to/output/${tf}_motif_peak_ids.txt
    awk -F'\t' 'NR==1{next} FNR==NR{ids[$1]=1; next} ids[$1] && $17!="" {print $1"\t"$17"\t"$10"\t"$8}' /path/to/output/${tf}_motif_peak_ids.txt annotated_peaks.txt > /path/to/output/${tf}_target_genes.txt
    echo "$tf done! $(wc -l < /path/to/output/${tf}_target_genes.txt) peaks with genes"
done
```

- Step 9 : Filter for promoter peaks only per TF :

```bash
for f in /path/to/output/*_target_genes.txt; do
    tf=$(basename $f _target_genes.txt)
    awk -F'\t' '$4~/[Pp]romoter/ {print $2}' $f | tr '|' '\n' | sort -u > /path/to/output/${tf}_promoter_genes.txt
    echo "$tf: $(wc -l < /path/to/output/${tf}_promoter_genes.txt) promoter genes"
done
```

- Step 10 : Find gene overlaps across multiple TFs :

```bash
# Genes shared by all 10 TFs
cat /path/to/output/*_promoter_genes.txt | sort | uniq -c | sort -rn | awk '$1==10 {print $2}' > all10_overlap_genes.txt

# Genes shared by 8 or more TFs
cat /path/to/output/*_promoter_genes.txt | sort | uniq -c | sort -rn | awk '$1>=8 {print $1"\t"$2}' > top_overlap_genes.txt
```

---

## De Novo Motifs Analysis

- Count de novo motifs found : `grep -c "^>" /path/to/homerMotifs.all.motifs`
- Check motif structure : `head -20 /path/to/homerMotifs.all.motifs`
- Check BestGuess TF match for individual motifs :
  - `head -1 /path/to/homerResults/motif1.motif`
  - `head -1 /path/to/homerResults/motif2.motif`
- Extract all BestGuess TF matches :

```bash
grep "^>" /path/to/homerResults/motif*.motif | grep "BestGuess" | \
sed 's/.*motif/motif/' | sed 's/.motif:>/\t/' | \
awk -F'\t' '{print $1"\t"$2}' | cut -f1,2 > denovo_bestguess.txt
```

---

## Interpreting Results

- p-value : raw statistical significance
- q-value (Benjamini) : p-value corrected for multiple testing — always use this, threshold q < 0.05
- % Target : percentage of your query peaks containing this motif
- % Background : percentage of background peaks containing this motif — a good hit has % Target > % Background
- Known motifs : compared against HOMER's database of known TF binding sites
- De novo motifs : discovered from scratch in your data — may reveal novel regulatory elements
