<div align="center">

# 🔬 RNA-Seq Differential Expression Pipeline
**Reference-based RNA-Seq analysis identifying genes differentially expressed upon *Pasilla* gene knockdown in *Drosophila melanogaster***

[![Galaxy](https://img.shields.io/badge/Workflow-Galaxy-1f6feb?style=flat-square&logo=galaxy&logoColor=white)](https://usegalaxy.org/)
[![DESeq2](https://img.shields.io/badge/DE%20Analysis-DESeq2-e11d48?style=flat-square)](https://bioconductor.org/packages/release/bioc/html/DESeq2.html)
[![STAR](https://img.shields.io/badge/Aligner-STAR-16a34a?style=flat-square)](https://github.com/alexdobin/STAR)
[![Organism](https://img.shields.io/badge/Organism-D.%20melanogaster%20dm6-f59e0b?style=flat-square)](https://www.ncbi.nlm.nih.gov/datasets/taxonomy/7227/)
[![Samples](https://img.shields.io/badge/Samples-7%20(3%20treated%20%2B%204%20untreated)-7c3aed?style=flat-square)]()

[Overview](#overview) · [Dataset](#dataset) · [Pipeline](#pipeline) · [Tools & Outputs](#tools--outputs) · [Key Results](#key-results) · [References](#references)

</div>

---

## Overview

This pipeline performs a complete reference-based RNA-Seq differential expression analysis on *Drosophila melanogaster* data. The goal is to identify genes up- or down-regulated by RNAi-mediated knockdown of the **Pasilla (PS)** splicing factor gene. The analysis uses a two-factor design (treatment + sequencing type) to correctly account for a mixed single-end/paired-end dataset.

---

## Dataset

| Sample | Condition | Sequencing type | GEO ID |
|--------|-----------|----------------|--------|
| GSM461176 | Untreated | Single-end | [GSM461176](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSM461176) |
| GSM461177 | Untreated | Paired-end | [GSM461177](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSM461177) |
| GSM461178 | Untreated | Paired-end | [GSM461178](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSM461178) |
| GSM461179 | Treated | Single-end | [GSM461179](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSM461179) |
| GSM461180 | Treated | Paired-end | [GSM461180](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSM461180) |
| GSM461181 | Treated | Paired-end | [GSM461181](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSM461181) |
| GSM461182 | Untreated | Single-end | [GSM461182](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSM461182) |

**Reference genome:** *D. melanogaster* dm6  
**Annotation:** Ensembl BDGP6.32.109 (GTF)

---

## Pipeline

```
Raw FASTQ reads (paired-end & single-end)
        │
        ▼
[1] Quality Control (Falco + MultiQC)
        │
        ▼
[2] Adapter & Quality Trimming (Cutadapt)
        │
        ▼
[3] Spliced Alignment to dm6 (STAR)
        │
        ├── BAM files → visualization (IGV / JBrowse2)
        └── Strand coverage bedgraphs
        │
        ▼
[4] Strandness Estimation (Infer Experiment)
        │
        ▼
[5] Read Counting per Gene (featureCounts)
        │
        ▼
[6] Differential Expression Analysis (DESeq2)
        │   2-factor design: Treatment + Sequencing type
        │
        ▼
[7] Annotation of DE Results (gene symbols, coordinates)
        │
        ▼
[8] Functional Enrichment (goseq → GO + KEGG)
        │
        ▼
[9] Pathway Visualization (Pathview)
```

---

## Tools & Outputs

### Stage 1 — Quality Control

**Tools:** `Falco v1.2.4` → `MultiQC v1.27`

Falco generates per-sample QC reports; MultiQC aggregates them. Key flags to check: per-base quality drop at read ends, GC content distribution, duplication levels (expected to be high in RNA-Seq).

| Finding | Value |
|---------|-------|
| Read length | 37 bp |
| Sequences per sample | 10.6M (GSM461177), 12.3M (GSM461180) |
| Notable issue | GSM461180 reverse reads show quality drop at 3' end |

---

### Stage 2 — Trimming

**Tool:** `Cutadapt v5.2`

Paired-end mode — trims both mates together, discarding entire pairs if either read falls below length threshold after quality trimming. This preserves read-pair integrity.

| Parameter | Value |
|-----------|-------|
| Quality cutoff | Q20 |
| Minimum length | 20 bp |
| Mode | Paired-end collection |

| Sample | Pairs discarded |
|--------|----------------|
| GSM461177 (untreated) | 147,810 (1.4%) |
| GSM461180 (treated) | 1,101,875 (9.0%) |

---

### Stage 3 — Spliced Alignment

**Tool:** `RNA STAR v2.7.11b`

A spliced-aware aligner is required for eukaryotic RNA-Seq because reads derived from processed mRNAs span exon-exon junctions absent from the genomic reference. STAR identifies split reads spanning introns during alignment.

| Parameter | Value |
|-----------|-------|
| Reference | dm6 (built-in) |
| Annotation | BDGP6.32.109 UCSC GTF |
| Junction overhang | 36 bp (read length − 1) |
| Extra outputs | Per-gene read counts, strand coverage bedgraphs |

| Sample | Uniquely mapped |
|--------|----------------|
| GSM461177 (untreated) | >83% |
| GSM461180 (treated) | >79% |

> Mapping rates below 70% warrant investigation for contamination. Both samples are within acceptable range.

**Output:** BAM files (coordinate-sorted), strand-specific coverage bedgraphs, per-gene counts

#### 3a — Alignment Visualization (IGV + JBrowse2)

BAM outputs were inspected at `chr4:540,000–560,000` in both **IGV** and **JBrowse2** for both paired-end samples (GSM461177, GSM461180).

- Grey peaks = per-base read coverage
- Connecting arcs between reads = splice junction events (reads spanning introns)
- Sashimi plot numbers = reads supporting each junction

**Strandness check region:** `chr3R:9,445,000–9,448,000` (*ps* gene locus). First-of-pair strand coloring showed an even red/blue split → unstranded library confirmed.

#### 3b — Strand Coverage (pyGenomeTracks)

STAR bedgraph outputs visualized at `chr4:540,000–560,000`:

- **Strand 1 → blue**
- **Strand 2 → red**

Both strands show comparable coverage (~1.2–1.5×), confirming **unstranded** library. A stranded library would show signal on one strand only.

---

### Stage 4 — Strandness Estimation

**Tool:** `Infer Experiment v5.0.3` (RSeQC)

Subsamples 200,000 reads and compares their strand orientation against a BED12 gene model to determine library strandness. Required to correctly parameterize featureCounts.

**Result:** Library is **unstranded** — forward and reverse fractions both ~46%, consistent across both samples.

---

### Stage 5 — Read Counting

**Tool:** `featureCounts v2.1.1`

Counts reads overlapping annotated exons per gene. Paired-end reads counted as single fragments. A minimum mapping quality filter (MAPQ ≥ 10) excludes poorly mapped reads.

| Parameter | Value |
|-----------|-------|
| Feature type | exon |
| Gene identifier | gene_id |
| Strandness | Unstranded |
| Min MAPQ | 10 |
| Paired-end counting | Yes (fragments) |

**Assignment rate:** ~63% — acceptable; rates below 50% require investigation.  
**Top expressed gene:** `FBgn0284245` (~128,000 counts in untreated)

**Outputs:** Count table (gene × sample), gene length file (used by goseq)

---

### Stage 6 — Differential Expression

**Tool:** `DESeq2 v2.11.40`

Two-factor design accounts for both treatment effect and sequencing type variation. DESeq2 applies internal normalization for sequencing depth and library composition — superior to TPM/RPKM for between-sample comparisons.

| Factor | Levels |
|--------|--------|
| Treatment (primary) | treated, untreated |
| Sequencing (secondary) | PE, SE |

**PCA result:** PC1 separates by treatment, PC2 separates by sequencing type — confirms the two-factor design was appropriate with no hidden confounders.

**Sample-to-sample heatmap:** Samples cluster first by treatment then by sequencing type, consistent with PCA. Dark blue = similar expression profiles. No unexpected groupings.

**Outputs:** Normalized count matrix, DE results table (log2FC, p-value, padj), PCA plot, sample-to-sample distance heatmap, MA plot, dispersion plot

---

### Stage 7 — Annotation

**Tool:** `Annotate DESeq2 output v1.1.0`

Appends gene symbol, chromosome, start, end, strand, and feature type to the DESeq2 results table using the Ensembl GTF.

---

### Stage 8 — Functional Enrichment

**Tool:** `goseq v1.50.0`

Corrects for gene-length bias inherent in RNA-Seq when testing GO/KEGG enrichment — longer genes are more likely to be called DE simply due to higher read counts.

**Inputs:** DE gene boolean table + gene length file (from featureCounts)

#### GO Analysis

| Category | Over-represented (padj < 0.05) | Under-represented (padj < 0.05) |
|----------|-------------------------------|--------------------------------|
| Biological Process | 50 | 5 |
| Cellular Component | 5 | 2 |
| Molecular Function | 5 | 0 |
| **Total** | **60** | **7** |

#### KEGG Analysis

| Pathway ID | Description | Status |
|------------|-------------|--------|
| dme01100 | Metabolic pathways | Over-represented |
| dme00010 | Glycolysis / Gluconeogenesis | Over-represented |

---

### Stage 9 — Pathway Visualization

**Tool:** `Pathview v1.34.0`

Overlays log2 fold change values onto KEGG pathway diagrams. Gene nodes are colored by log2FC (green = down, red = up).

**Pathways visualized:** dme00010 (Glycolysis), dme03040 (Spliceosome)

---

## Key Results

| Metric | Value |
|--------|-------|
| Total genes tested | ~13,900 |
| Significantly DE genes (padj < 0.05) | 966 (4.04%) |
| DE genes with \|log2FC\| > 1 | 113 (11.79% of significant) |
| Pasilla gene (FBgn0261552) | Downregulated ✓ (padj < 0.05) |
| Most over-expressed gene | FBgn0025111 (*Ant2*, adenine nucleotide translocase 2) — chrX |
| Most over-represented GO term | RNA splicing / mRNA processing (Biological Process) |
| Most significant KEGG pathway | dme00010 Glycolysis / Gluconeogenesis |

---

## References

- Dobin et al. (2013). STAR: ultrafast universal RNA-seq aligner. *Bioinformatics*
- Liao et al. (2013). featureCounts: an efficient general purpose program. *Bioinformatics*
- Love et al. (2014). Moderated estimation of fold change and dispersion with DESeq2. *Genome Biology*
- Young et al. (2010). Gene ontology analysis for RNA-seq: accounting for selection bias. *Genome Biology*
- Luo & Brouwer (2013). Pathview: an R/Bioconductor package for pathway-based data integration. *Bioinformatics*
- Wang et al. (2012). RSeQC: quality control of RNA-seq experiments. *Bioinformatics*
