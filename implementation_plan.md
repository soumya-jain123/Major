# 🏥 Implementation Plan — Major Project
## "A Multimodal Retrieval-Augmented Generation Framework for Automated Chest X-Ray Report Generation"
> **University:** UPES Dehradun, School of Computer Science
> **Mentor:** Dr. Anish Kumar Vishwakarma (Associate Professor SG, AI Cluster)
> **Deadline:** Presentation after 10th October 2026 | **Team:** 3 Members

---

## 📋 Key Findings from All Documents

### From `synopsis_report.docx` (Formal Synopsis)

| Section | Key Content |
|---|---|
| **Project Title** | A Multimodal RAG Framework for Automated Chest X-Ray Report Generation |
| **Primary Dataset** | **MIMIC-CXR** (chest X-rays + free-text reports, multi-view + longitudinal) |
| **Secondary Dataset** | IU X-Ray (Open-i) — for early prototyping and cross-dataset comparison |
| **CheXpert Plus** | Used for EDA and supplementary embedding/label data |
| **Output Format** | Structured: `Findings` followed by `Impression` |
| **Primary Evidence** | Current study always dominates; historical/retrieved = supporting context only |
| **Evidence Priority** | Current visual > Historical (genuine) > Retrieved similar cases > Clinical knowledge |
| **Evaluation Levels** | 4 levels: Retrieval quality, Language quality, Clinical correctness, Factuality |
| **Key Papers** | STREAM (Yang et al., 2025), RA-RRG (Park et al., 2026), X-REM (Jeong et al., 2024), MAIRA-1 (Hyland et al., 2024), CXR-LLAVA (Lee et al., 2023) |

### 6 Formal Objectives (from Synopsis)

1. Design a multimodal RAG framework for current + historical multi-view CXR + reports
2. Develop a **contrastive image-text alignment module** (CLIP/InfoNCE) for joint embedding space
3. Implement **spatial fusion** (multi-view) and **temporal fusion** (historical studies)
4. Construct a **vector-indexed retrieval store** (FAISS) for prior cases + evidence
5. Generate a **grounded Findings + Impression report** via LLM with RAG, avoiding unsupported claims
6. Evaluate **retrieval quality, language quality, clinical correctness, factuality** + ablation experiments

### Research Gaps Identified in Synopsis

- Retrieval not jointly conditioned on anatomy, view, and clinical context
- Longitudinal retrieval risks patient-level leakage if splits are not carefully maintained
- Rare findings underrepresented → limited generalization
- Cross-institution robustness not established
- Factuality evaluation inconsistent; lexical metrics alone insufficient

### Technical Concepts Required (from Synopsis)

| Concept | Description |
|---|---|
| Vision Transformers | ViT/Swin-style backbone adapted for chest radiographs |
| Contrastive Learning | CLIP/InfoNCE — align matching image-report pairs |
| Dense Retrieval | FAISS vector search for approximate nearest-neighbour |
| RAG | Retrieved evidence constrains LLM generation |
| Spatial-Temporal Fusion | Multi-view + historical study fusion |
| Clinical Evaluation | CheXbert F1, RadGraph F1, factuality/hallucination analysis |
| Language Evaluation | BLEU, ROUGE-L, METEOR, BERTScore |

---

### From `Complete_STREAM_Multimodal_RAG_Methodology.docx`

| Detail | Content |
|---|---|
| **Encoder candidates** | BioViL-T (temporal), MedCLIP (image-text alignment) |
| **Text encoder** | BioLinkBERT, ClinicalBERT |
| **Baselines** | A: No retrieval, B: Independent embeddings, C: Joint contrastive, D: +RAG, E: +Multiview, F: +Temporal, Full |
| **Fusion** | Concatenate or attention-fuse view representations; fallback if single-view |
| **Hard negatives** | Visually similar cases with different findings for contrastive training |

---

### From EDA Code (CheXpert Plus — already completed)

| Finding | Implementation Decision |
|---|---|
| `section_impression` = 99.93% coverage | **Primary text target** for encoding and generation |
| `section_findings` = only 26.6% | Always fallback to `section_impression` |
| Reports avg 246 tokens; 0.85% > 512 | Single-chunk FAISS indexing — no chunking needed |
| 4.19% reports have `___` de-id artifacts | Normalize to `[DEID]` before embedding |
| 85.5% frontal images | Start frontal-only; add lateral in Phase 2 |
| 14 pathology labels, severe class imbalance | Use weighted loss / sampling in contrastive training |
| Exact duplicate reports: 4 | Remove before indexing |
| Official valid = 234 images only | Must create own 80/10/10 patient-level split |
| Zero cross-split patient leakage | CheXpert Plus splits are clean |

---

## Two-Pipeline Architecture (from Synopsis)

```
===============================================================
PIPELINE 1: Knowledge Base Construction
===============================================================

MIMIC-CXR + IU X-Ray (training studies)
              |
              v
     Study-Level Organization
     [patient_id, study_id, views[], report,
      study_date, prior_study_links]
              |
              v
     Image Preprocessing
     (resize, normalize, view tagging)
              |
              v
  Medical Vision Transformer (BioViL-T / MedCLIP)
              |
     +---------+---------+
     v                   v
 View Embeddings   Temporal Embeddings
     +----+----+
          |
          v
  Spatial-Temporal Fusion
  (concat/attention; fallback if single-view)
          |
          v
    Visual Embedding
          |                    Radiology Reports
          |                (Findings + Impression)
          |                         |
          |                         v
          |              Medical Language Encoder
          |              (BioLinkBERT / ClinicalBERT)
          |                         |
          |                         v
          |                   Text Embedding
          |                         |
          +----------+--------------+
                     |
                     v
         Contrastive Learning (CLIP/InfoNCE)
         [+ Hard negatives: visually similar,
           clinically different cases]
                     |
                     v
           Joint Multimodal Embedding Space
                     |
                     v
          FAISS Vector Index + Metadata
          [study_id, view_type, impression,
           findings, pathology_labels, temporal_info]

===============================================================
PIPELINE 2: Retrieval + RAG Report Generation
===============================================================

New Patient Study (test time)
              |
              v
     Image Preprocessing + Encoding
              |
              v
     Multimodal Query Embedding
              |
              v
  FAISS Similarity Search (Top-K)
              |
              v
   Retrieved Evidence:
   - Top-K similar study reports
   - Anatomical / clinical context
   - Historical prior reports (if available)
              |
              v
   RAG Prompt Construction
   [Priority: Current > Historical > Retrieved > Knowledge]
              |
              v
   LLM (Mistral-7B / GPT-4o-mini)
              |
              v
   Structured Report:
   Findings: ...
   Impression: ...
              |
              v
   Evaluation (4 Levels):
   - Retrieval: Recall@K, Precision@K, MRR
   - Language: BLEU, ROUGE-L, METEOR, BERTScore
   - Clinical: CheXbert F1, RadGraph F1
   - Factuality: Unsupported-claim analysis
```

---

## 👥 Team of 3 — Division of Work

> Primary ownership below. All 3 collaborate on integration, evaluation runs, and the final presentation.

---

### Person A — Data Pipeline + Vision Encoder + Contrastive Learning

**Role:** ML Engineer (Vision + Training)

#### Phase 1: Data Pipeline (Days 1–4)
- [ ] Access MIMIC-CXR (PhysioNet) — download CSVs + images (or a 20K-study subset)
- [ ] Also keep CheXpert Plus as supplementary (already EDA'd)
- [ ] Build study-level records: `patient_id`, `study_id`, `image_paths[]`, `view_types[]`, `findings`, `impression`, `study_date`, `prior_study_ids[]`
- [ ] Apply patient-level 80/10/10 train/val/test split — strictly no patient leakage across splits
- [ ] De-id normalization: replace `___` with `[DEID]` in all text
- [ ] Remove exact duplicate reports (4 confirmed in CheXpert Plus)
- [ ] Handle sparsity: if `section_findings` absent → use `section_impression` as text target
- [ ] Save clean `dataset_train.csv`, `dataset_val.csv`, `dataset_test.csv`

#### Phase 2: Image Preprocessing (Days 3–5, overlap)
- [ ] Resize to 224x224 (or 512x512 per encoder requirement)
- [ ] Normalize pixel intensities per encoder spec
- [ ] Build PyTorch `ChestXRayDataset` class
- [ ] Tag each image: `frontal` vs `lateral`, `AP` vs `PA`
- [ ] Validate loading success rate; log failures

#### Phase 3: Medical Vision Encoder (Days 5–8)
- [ ] Select encoder: **BioViL-T** (preferred — supports temporal) or **MedCLIP** (simpler fallback)
- [ ] Load pretrained weights from HuggingFace
- [ ] Batch inference → extract embeddings, save as `.npy` / HDF5
- [ ] If multiple views in one study: keep separately before fusion step

#### Phase 4: Contrastive Learning Training (Days 7–11)
- [ ] Build paired dataset: `(fused_image_embedding, text_embedding)` per study
- [ ] Implement InfoNCE / NT-Xent loss
- [ ] Generate hard negatives: visually similar studies with different pathology labels
- [ ] Train projection heads (image → joint space, text → joint space)
- [ ] Tune: temperature, batch size, learning rate
- [ ] Evaluate: image-to-text Recall@K on val set
- [ ] Save best checkpoint

**Deliverable by Oct 9:** Contrastive-trained joint embedding model + Recall@K metric

---

### Person B — Text Encoder + FAISS + RAG Pipeline

**Role:** ML Engineer (NLP + Retrieval)

#### Phase 1: Medical Text Encoder (Days 1–5)
- [ ] Select encoder: **BioLinkBERT-large** or **ClinicalBERT** or **RadBERT**
- [ ] Encode `section_impression` (fallback: `section_findings`) for all training records
- [ ] Batch encode → save embeddings to disk
- [ ] Verify 246-token average aligns with encoder's max length (512)

#### Phase 2: Spatial-Temporal Fusion (Days 4–6, coordinate with A)
- [ ] View fusion: concat frontal + lateral embeddings → project to shared dim
- [ ] Graceful fallback if only frontal available
- [ ] Temporal context: use `patient_report_date_order` / `study_date` to tag historical studies
- [ ] Historical study embedding = auxiliary context (not primary)

#### Phase 3: FAISS Vector DB (Days 5–8)
- [ ] Install `faiss-cpu` (or `faiss-gpu` if GPU available)
- [ ] Index joint embeddings (post-contrastive, coordinate with A)
- [ ] Metadata per indexed item: `study_id`, `patient_id`, `split`, `impression`, `findings`, `pathology_labels`, `view_type`, `study_date`
- [ ] Enforce: test partition NEVER indexed — strict retrieval leakage prevention
- [ ] Implement top-K similarity search function
- [ ] Evaluate retrieval independently: Recall@K, Precision@K, MRR on val set

#### Phase 4: RAG Prompt + LLM (Days 7–10)
- [ ] Choose LLM: **Mistral-7B-Instruct** via Ollama (local, free) or **GPT-4o-mini** (API, capped)
- [ ] Design structured prompt:
  ```
  SYSTEM: You are a radiology report assistant. Generate only what is supported by evidence.
  
  CURRENT STUDY CONTEXT: [encoded visual summary / pathology labels]
  
  RETRIEVED SIMILAR CASES (supporting evidence):
  Case 1: [impression from retrieved study]
  Case 2: [impression from retrieved study]
  Case 3: [impression from retrieved study]
  
  HISTORICAL CONTEXT (if available): [prior study impression, labeled with date]
  
  INSTRUCTION: Generate a structured report.
  Output ONLY:
  Findings: ...
  Impression: ...
  Do not invent findings not supported by the current study.
  ```
- [ ] Implement full RAG pipeline: query → retrieve → prompt → generate
- [ ] Use deterministic generation (greedy / temp=0) for reproducibility
- [ ] Test qualitatively on 20–30 validation cases

**Deliverable by Oct 9:** End-to-end RAG pipeline producing Findings + Impression

---

### Person C — Evaluation Framework + EDA Wrap-up + Presentation

**Role:** Research Engineer (Evaluation + Documentation)

#### Phase 1: EDA Review & Dataset Documentation (Days 1–3)
- [ ] Review all existing EDA notebooks (`chexpert_plus_core_table_eda.ipynb`, etc.)
- [ ] Extract key statistics for slides (223K records, 64K patients, section coverage, pathology distribution)
- [ ] Document dataset decision: MIMIC-CXR (primary) vs CheXpert Plus (supplementary EDA)
- [ ] Finalize label strategy: `report.csv` for recall, `impression.csv` for precision

#### Phase 2: Evaluation Framework (Days 3–8)
- [ ] **Language metrics:**
  - BLEU-1/2/4 (`sacrebleu`)
  - ROUGE-L (`rouge-score`)
  - METEOR
  - BERTScore (`bert-score` library)
- [ ] **Clinical metrics:**
  - CheXbert F1 (run CheXbert on generated vs. reference → compare 14-label vectors)
  - RadGraph F1 (entity + relation overlap) — if time permits
- [ ] **Retrieval metrics:**
  - Recall@K (K=1,5,10)
  - Precision@K
  - Mean Reciprocal Rank (MRR)
- [ ] **Factuality:**
  - Count sentences with no retrieval/current-study grounding
  - Simple keyword-based unsupported claim detector
- [ ] Build evaluation runner: `evaluate(generated_report, reference_report)` → score dict
- [ ] Build retrieval evaluator: `evaluate_retrieval(query_embedding, relevant_study_ids)` → metrics

#### Phase 3: Baseline & Ablation Experiments (Days 7–10)
Run on the same val/test set, same protocol:

| Experiment | Config | Expected Metric |
|---|---|---|
| Baseline A | Vision encoder → LLM (no retrieval) | Lowest BLEU/clinical scores |
| Baseline B | Independent embeddings + retrieval | Better retrieval, mid scores |
| Experiment C | Joint contrastive + retrieval | Improved Recall@K |
| Experiment D | C + RAG | Improved BLEU/CheXbert F1 |
| Experiment E | D + multi-view fusion | Marginal improvement |
| Experiment F | E + temporal context | Improvement on longitudinal cases |
| **Full Model** | All components | Best clinical + factuality scores |

#### Phase 4: Presentation & Report (Days 9–11)
- [ ] Compile all metric results into comparison table
- [ ] Build architecture diagram (2 pipelines: KB construction + RAG generation)
- [ ] Prepare live demo: pick 3–5 test X-rays → show retrieved cases + generated report
- [ ] Write report sections: Introduction, Methodology, Results, Discussion, Conclusion
- [ ] Prepare presentation slides (10–15 slides)

**Deliverable by Oct 10:** Full evaluation table + demo + slides

---

## 📅 Day-by-Day Gantt (Sep 28 → Oct 10)

| Day | Date | Person A | Person B | Person C |
|---|---|---|---|---|
| 1 | Sep 28 | MIMIC-CXR download + CSV cleaning | Text encoder setup (BioLinkBERT) | EDA review + slides template |
| 2 | Sep 29 | Study-level table building + patient splits | Encode impressions (100 samples test) | Dataset decision doc + label strategy |
| 3 | Sep 30 | De-id normalization + split verification | View fusion design | Evaluation framework scaffold |
| 4 | Oct 1 | Image preprocessing + PyTorch Dataset | FAISS setup + indexing plan | BLEU + ROUGE-L implementation |
| 5 | Oct 2 | BioViL-T pretrained inference | Retrieval function + top-K test | METEOR + BERTScore |
| 6 | Oct 3 | Embedding extraction + save to disk | Retrieval quality eval (Recall@K, MRR) | CheXbert F1 setup |
| 7 | Oct 4 | Contrastive training begins | RAG prompt design + LLM setup | Evaluation runner complete + Baseline A |
| 8 | Oct 5 | Training + hard negative generation | LLM integration + test 20 cases | Baseline B eval |
| 9 | Oct 6 | Checkpoint eval + Recall@K on val | Full RAG pipeline end-to-end | Experiments C + D eval |
| 10 | Oct 7 | Integration + embedding handoff to B | Integration + FAISS reindex w/ joint emb | Demo preparation (3–5 test cases) |
| 11 | Oct 8 | Bug fixes + tuning | Bug fixes + generation quality check | Slides + comparison table |
| 12 | Oct 9 | Final model checkpoint | Final RAG pipeline | Presentation polish + rehearsal |
| 13 | Oct 10 | **PRESENTATION** | **PRESENTATION** | **PRESENTATION** |

---

## 🔧 Recommended Tech Stack

| Component | Recommended Tool |
|---|---|
| Primary Dataset | MIMIC-CXR (PhysioNet) + CheXpert Plus (supplementary) |
| Secondary Dataset | IU X-Ray (Open-i) for early prototyping |
| Vision Encoder | `BioViL-T` (HuggingFace: `microsoft/BioViL-T`) |
| Text Encoder | `BioLinkBERT-large` or `emilyalsentzer/Bio_ClinicalBERT` |
| Contrastive Training | PyTorch + custom InfoNCE / NT-Xent |
| Vector DB | `faiss-cpu` / `faiss-gpu` |
| LLM | `Mistral-7B-Instruct-v0.3` via Ollama (local) or `GPT-4o-mini` |
| Language Eval | `sacrebleu`, `rouge-score`, `bert-score`, `nltk` (METEOR) |
| Clinical Eval | `CheXbert` (Stanford NLP) + `RadGraph` (if time allows) |
| Experiment Tracking | `wandb` (free tier) |
| Data | `pandas`, `Pillow`, `pydicom`, `torchvision` |
| Notebooks | Jupyter (reuse existing CheXpert EDA notebooks) |

---

## 📊 Evaluation Framework Summary (from Synopsis)

### Retrieval Metrics (independent of generation)
- **Recall@K** (K = 1, 5, 10) — are relevant studies returned?
- **Precision@K** — how many returned results are relevant?
- **MRR (Mean Reciprocal Rank)** — how highly ranked is the first relevant result?

### Language Metrics
- **BLEU-1/2/4** — n-gram precision vs. reference report
- **ROUGE-L** — longest common subsequence match
- **METEOR** — alignment + synonym matching
- **BERTScore** — semantic similarity in embedding space

### Clinical Metrics
- **CheXbert F1** — pathology label agreement (14 labels) between generated and reference
- **RadGraph F1** — entity and relation overlap (if time permits)

### Factuality
- Unsupported-claim count — generated sentences not traceable to current study or retrieved evidence

---

## 🎯 Minimum Viable Demo (for Oct 10)

| # | Component | Status Target |
|---|---|---|
| 1 | Clean dataset with splits (MIMIC-CXR or CheXpert Plus) | ✅ Must have |
| 2 | Image embeddings from pretrained BioViL-T/MedCLIP | ✅ Must have |
| 3 | Text embeddings from BioLinkBERT on impressions | ✅ Must have |
| 4 | FAISS retrieval: top-3 similar cases for a test X-ray | ✅ Must have |
| 5 | LLM generates Findings + Impression from retrieved context | ✅ Must have |
| 6 | BLEU + ROUGE + CheXbert F1 numbers vs. reference | ✅ Must have |
| 7 | Contrastive fine-tuning (Recall@K improvement) | 🔄 Show progress / preliminary numbers |
| 8 | Multi-view and temporal fusion ablation | 🔄 Stretch goal |

> [!TIP]
> For the demo: pick 3 test X-rays with different pathology profiles (e.g., Pleural Effusion, Pneumothorax, No Finding). Show: (1) the image, (2) top-3 retrieved impressions, (3) the LLM-generated report, (4) the reference report, (5) the metric scores.

---

## ⚠️ Risks & Mitigations

| Risk | Mitigation |
|---|---|
| MIMIC-CXR access takes time (PhysioNet approval) | Start with CheXpert Plus (already EDA'd) as primary — switch when MIMIC ready |
| Image download too slow / large | Use 20K-study frontal-only subset for prototype |
| Contrastive training takes > 4 days | Present pretrained-encoder-only retrieval as Baseline B; show improvement trend |
| LLM API costs | Use Mistral-7B locally via Ollama (free, runs on 8GB VRAM) |
| `section_findings` absent in 73% of CheXpert records | Always fallback to `section_impression` (99.93% coverage) |
| Patient leakage in longitudinal retrieval | Strictly never index val/test records; enforce at FAISS build time |
| CheXbert setup complexity | Pre-compute CheXbert labels offline; load pre-extracted label files for eval |
| RadGraph setup too complex for timeline | Skip RadGraph F1; focus on CheXbert F1 + BERTScore as clinical proxy |

---

## 🚀 Immediate Next Steps (Today, Sep 28)

> [!IMPORTANT]
> These must be started today.

1. **All 3:** Create shared GitHub repo + Google Drive folder for data, checkpoints, and results
2. **Person A:** Apply for MIMIC-CXR on PhysioNet (takes 1–3 days for approval); meanwhile use CheXpert Plus to build the pipeline
3. **Person B:** `pip install transformers faiss-cpu sentence-transformers sacrebleu rouge-score bert-score`; test BioLinkBERT encoding on 100 rows from `impression.csv`
4. **Person C:** Open all 5 EDA notebooks; extract top-10 statistics for the presentation deck; set up CheXbert evaluation script

---

## 📚 Key References to Know

| Paper | Why Important |
|---|---|
| Yang et al. (2025) — **STREAM** | Spatio-temporal + retrieval for CXR generation — direct inspiration |
| Park et al. (2026) — **RA-RRG** | Key-phrase retrieval + multi-view aggregation |
| Jeong et al. (2024) — **X-REM** | Image-text matching improves retrieval vs. image-only |
| Hyland et al. (2024) — **MAIRA-1** | Domain-specific encoder + LM for radiology |
| Ranjit et al. (2023) | RAG with OpenAI models for CXR reports |
| Yu et al. (2023) | Evaluation gap: lexical metrics vs. clinical correctness |

---

*Plan generated: 2026-09-28*
*Sources: `synopsis_report.docx`, `Complete_STREAM_Multimodal_RAG_Methodology.docx`, `chexpert_plus_eda_report.md`, `chexpert_plus_repositories.md`, `df_chexpert_plus_240401_metadata.json`*
