# CheXpert Plus: Useful GitHub Repositories

This document summarizes repositories that are relevant to **CheXpert Plus**, especially for work involving **chest X-rays, Findings, Impression, CheXbert labels, RadGraph annotations, multimodal learning, EDA, and preprocessing**.

---

## 1. Stanford-AIMI / chexpert-plus

**Repository:** https://github.com/Stanford-AIMI/chexpert-plus

**Organization:** Stanford AIMI / Stanford ML Group

### About the repository

This is the **official CheXpert Plus repository** from the Stanford team behind the dataset.

It contains code, documentation, and tutorials for working with the CheXpert Plus release, including its metadata and report annotations.

### Useful CheXpert Plus components

The repository is relevant to:

- `df_chexpert_plus_240401.csv`
- Report sections such as:
  - Findings
  - Impression
- CheXbert pathology labels
- RadGraph-XL annotations
- Image/report mappings
- Report-generation tutorials
- Retrieval of annotations associated with individual studies/images

### Purpose

Use this repository as the **primary reference for the official CheXpert Plus data structure and annotation format**.

It is the best starting point when you need to understand:

```text
Image
  ↓
Study
  ↓
Report
  ├── Findings
  └── Impression
  ↓
CheXbert labels
  ↓
RadGraph entities / relations
```

### Why it is useful for EDA

For your EDA, this repository is particularly useful for understanding:

- How `section_findings` is represented
- How `section_impression` is represented
- How pathology labels are associated with reports
- How RadGraph annotations can be connected to report text
- How image/report relationships are represented

### Recommendation

**Start here first.** Treat this repository as the authoritative implementation/reference for CheXpert Plus.

---

## 2. StevenSong / cxr-data-ingest

**Repository:** https://github.com/StevenSong/cxr-data-ingest

### About the repository

This repository focuses on **preparing and organizing chest X-ray datasets for research**.

It includes a dedicated CheXpert Plus processing workflow.

One particularly relevant component is:

```text
2_prepare_chexpertplus_metadata.ipynb
```

### Purpose

The CheXpert Plus pipeline can be useful for turning the original release into a cleaner research-oriented structure.

The repository includes processing related to:

- Metadata preparation
- Study IDs
- Report processing
- Findings/Impression extraction
- Report deduplication
- Dataset organization
- Train/validation/test preparation
- CheXbert-related report preprocessing

### Why it is useful for your project

This repository is valuable after understanding the official format because it provides ideas for:

```text
Raw CheXpert Plus
       ↓
Metadata processing
       ↓
Report section extraction
       ↓
Deduplication / cleaning
       ↓
Research-ready tables
       ↓
EDA / ML
```

### Why it is especially relevant to EDA

It can help when building a clean dataset containing fields such as:

- Patient ID
- Study ID
- Image path
- Findings
- Impression
- Labels
- Split information

### Recommendation

Use this repository as a **preprocessing/data-engineering reference** after reviewing the official Stanford repository.

---

## 3. rajpurkarlab / ReXrank

**Repository:** https://github.com/rajpurkarlab/ReXrank

### About the repository

ReXrank is a radiology report evaluation/research project from the Rajpurkar Lab ecosystem.

It works with chest X-ray reports and uses CheXpert Plus data.

### Important fields

The project makes use of report structures including:

```text
section_findings
section_impression
reports
frontal_lateral
```

This is particularly useful for understanding how CheXpert Plus is consumed by downstream radiology research.

### Purpose

It is useful for:

- Radiology report analysis
- Report generation
- Evaluation of generated reports
- Findings/Impression-based evaluation
- Image-report research

### Why it is useful for you

Your project specifically involves **Findings and Impression**, so this repository provides a practical example of how downstream research handles those sections.

Conceptually:

```text
X-ray
  ↓
Findings
  ↓
Impression
  ↓
Generated / evaluated report
```

### Recommendation

Use ReXrank as a **downstream research reference** for understanding how Findings and Impression are used in modern radiology report generation/evaluation.

---

## 4. rajpurkarlab / ReXrank-metric

**Repository:** https://github.com/rajpurkarlab/ReXrank-metric

### About the repository

This repository is associated with the ReXrank evaluation ecosystem and focuses on evaluating radiology reports.

### Purpose

It can be useful for studying:

- Reference report evaluation
- Findings-based evaluation
- Impression-based evaluation
- Automatic radiology report comparison

### Why it is relevant

Your CheXpert Plus analysis may eventually move beyond EDA toward:

```text
Reference Findings
        ↕
Generated Findings

Reference Impression
        ↕
Generated Impression
```

This type of evaluation is useful when developing an image-to-report or multimodal system.

### Recommendation

Use this repository when moving from **EDA into report-generation evaluation**.

---

## 5. Stanford CheXpert Plus RRG Tutorials

**Repository:** https://github.com/Stanford-AIMI/chexpert-plus/tree/main/tutorials/RRG

### About the repository

This is part of the official Stanford CheXpert Plus repository and contains **Radiology Report Generation (RRG)** tutorials.

### Purpose

It demonstrates how CheXpert Plus can be used for image-to-report generation.

The official material includes workflows/models for report generation involving:

- CheXpert Findings
- CheXpert Impression
- MIMIC-CXR Findings
- MIMIC-CXR Impression

### Why it is useful

This is particularly relevant because Stanford treats **Findings and Impression as meaningful, distinct report-generation targets**.

The basic idea is:

```text
Chest X-ray
    ↓
Vision Encoder
    ↓
Text Decoder
    ↓
Findings / Impression
```

### Why it matters for your EDA

Before training a multimodal or report-generation model, you can investigate:

- Findings length
- Impression length
- Vocabulary
- Medical entities
- Pathology mentions
- Section availability
- Findings → Impression relationships

### Recommendation

Use this as the bridge between **EDA and model development**.

---

## 6. uzh-dqbm-cmi / RadVLM

**Repository:** https://github.com/uzh-dqbm-cmi/RadVLM

### About the repository

RadVLM is a broader radiology vision-language research project.

It supports multimodal medical imaging/report use cases and works with multiple radiology datasets.

### Purpose

It is useful for investigating:

- Vision-language models
- Medical image-text learning
- Radiology report understanding
- Multimodal representation learning
- Image + text fusion

### Why it is relevant to CheXpert Plus

CheXpert Plus is naturally suited to multimodal research because it links:

```text
Chest X-ray
        +
Radiology report
```

and the report can contain:

```text
Findings
Impression
Other sections
```

### Recommendation

Use RadVLM as an **architecture/multimodal-learning reference**, rather than as the primary CheXpert Plus data reference.

---

## 7. Levi-ZJY / TemMed-Bench

**Repository:** https://github.com/Levi-ZJY/TemMed-Bench

### About the repository

TemMed-Bench is a broader medical vision-language benchmark that includes CheXpert Plus as one of its data sources.

### Purpose

It demonstrates how CheXpert Plus data can be transformed into resources for tasks such as:

- Image-text learning
- Vision-language evaluation
- Report/image retrieval
- Medical VQA
- Multimodal benchmarking

### Why it is useful

It is useful if you want to understand how other projects convert CheXpert Plus into:

```text
Image + Report
Image + Question
Image + Answer
Image + Clinical information
```

### Recommendation

Use this as an **example of downstream multimodal dataset construction**.

---

# Repository Comparison

| Repository | Main Purpose | CheXpert Plus Relevance | Best Use |
|---|---|---|---|
| [Stanford-AIMI/chexpert-plus](https://github.com/Stanford-AIMI/chexpert-plus) | Official dataset/code | Very High | Data structure, annotations, reports |
| [cxr-data-ingest](https://github.com/StevenSong/cxr-data-ingest) | Dataset preprocessing | High | Cleaning and metadata preparation |
| [ReXrank](https://github.com/rajpurkarlab/ReXrank) | Report research/evaluation | High | Findings/Impression usage |
| [ReXrank-metric](https://github.com/rajpurkarlab/ReXrank-metric) | Report evaluation | Medium-High | Report comparison |
| [CheXpert Plus RRG](https://github.com/Stanford-AIMI/chexpert-plus/tree/main/tutorials/RRG) | Report generation | Very High | Findings/Impression modeling |
| [RadVLM](https://github.com/uzh-dqbm-cmi/RadVLM) | Radiology vision-language models | Medium | Multimodal architecture |
| [TemMed-Bench](https://github.com/Levi-ZJY/TemMed-Bench) | Medical VLM benchmark | Medium | Multimodal dataset construction |

---

# Which Repository Should You Use for What?

## For understanding the raw CheXpert Plus dataset

Use:

**Stanford-AIMI/chexpert-plus**

---

## For preparing the dataset before EDA

Use:

**cxr-data-ingest**

---

## For analyzing Findings and Impression

Use:

**Stanford CheXpert Plus RRG**

and

**ReXrank**

---

## For report-generation research

Use:

**Stanford CheXpert Plus RRG**

and

**ReXrank**

---

## For multimodal model design

Use:

**RadVLM**

and

**TemMed-Bench**

---

# Recommended Workflow for Your Project

Since you have **CheXpert Plus with X-rays + reports**, a practical workflow is:

```text
                         CheXpert Plus
                              │
                              ▼
               Stanford-AIMI/chexpert-plus
                              │
                Understand official format
                              │
                              ▼
                     cxr-data-ingest
                              │
                   Clean / organize data
                              │
                              ▼
                            EDA
              ┌───────────────┼───────────────┐
              ↓               ↓               ↓
           Images          Findings       Impression
              │               │               │
              └───────────────┼───────────────┘
                              ↓
                       Pathology labels
                              │
                              ↓
                  Findings ↔ Impression
                              │
                              ↓
                    Image ↔ Report EDA
                              │
                              ▼
                   Multimodal Modeling
                              │
                 ┌────────────┴────────────┐
                 ↓                         ↓
             RRG / ReXrank              RadVLM
```

---

# Important CheXpert Plus Data Caveat

When connecting separate annotation files to the main CheXpert Plus CSV, **do not assume that row positions always correspond directly across files**.

For example, the official repository has had discussion around differences in sample counts between the main CSV and some report annotation files. A robust pipeline should therefore use the dataset's documented identifiers/mappings rather than blindly joining by row number.

For research work, always verify:

```text
Patient ID
Study ID
Image path
Report ID / mapping
```

before performing multimodal analysis.

---

# Suggested EDA Focus for Findings + Impression

For your specific project, the most useful EDA would be:

### Image EDA
- Image dimensions
- Aspect ratio
- Frontal vs lateral
- AP vs PA
- Pixel intensity
- Image quality
- Multiple images per study

### Findings EDA
- Availability
- Word/token length
- Sentence length
- Medical vocabulary
- Disease mentions
- Negation
- Uncertainty
- RadGraph entities/relations

### Impression EDA
- Availability
- Word/token length
- Medical vocabulary
- Pathology mentions
- Negation
- Uncertainty
- RadGraph entities/relations

### Findings ↔ Impression
- Semantic similarity
- Shared pathologies
- Findings-only pathologies
- Impression-only pathologies
- Compression ratio
- Entity overlap

### Image ↔ Report
- Pathology/image association
- Report completeness
- Study/image counts
- Multi-view studies
- Patient-level duplication
- Potential leakage

---

## Key Links

- **Official CheXpert Plus:** https://github.com/Stanford-AIMI/chexpert-plus
- **CXR Data Ingest:** https://github.com/StevenSong/cxr-data-ingest
- **ReXrank:** https://github.com/rajpurkarlab/ReXrank
- **ReXrank Metric:** https://github.com/rajpurkarlab/ReXrank-metric
- **CheXpert Plus RRG Tutorials:** https://github.com/Stanford-AIMI/chexpert-plus/tree/main/tutorials/RRG
- **RadVLM:** https://github.com/uzh-dqbm-cmi/RadVLM
- **TemMed-Bench:** https://github.com/Levi-ZJY/TemMed-Bench

---

## Recommended Starting Order

```text
1. Stanford-AIMI/chexpert-plus
2. cxr-data-ingest
3. CheXpert Plus RRG
4. ReXrank
5. ReXrank-metric
6. RadVLM
7. TemMed-Bench
```

This order moves from **official dataset understanding → preprocessing → EDA → report generation/evaluation → multimodal modeling**.
