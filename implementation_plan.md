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
| **Primary Dataset** | **CheXpert** (with the report-bearing CheXpert Plus release used where paired report text is required) |
| **Secondary Dataset** | None planned initially — keep the pipeline centered on CheXpert; external datasets are optional future validation only |
| **CheXpert / CheXpert Plus** | Primary image + pathology-label source; use the paired report fields from the report-bearing release for report generation |
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
| Zero cross-split patient leakage | CheXpert Plus splits are clean; still recreate and verify our own patient-level 80/10/10 split for the final experiment |

> **Dataset decision:** The implementation is now centered on **CheXpert**. Because automated report generation requires paired free-text references, the report-bearing **CheXpert Plus** release will be used whenever the experiment requires `Findings`/`Impression` text. If only the original label-only CheXpert release is available, report-generation metrics such as BLEU/ROUGE/BERTScore cannot be computed against reference reports; in that case the project must switch to label-grounded generation/evaluation rather than silently treating labels as reports.

---

## Two-Pipeline Architecture (from Synopsis)

```
===============================================================
PIPELINE 1: Knowledge Base Construction
===============================================================

CheXpert (report-bearing CheXpert Plus release where applicable)
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
 View Embeddings   Study Metadata
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
   - Prior reports only if the selected CheXpert release contains valid longitudinal links
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

#### Phase 1: Data Pipeline (Days 1–2)
- [ ] Prepare the CheXpert image/label tables and paired report metadata; use a manageable study subset for the first end-to-end run
- [ ] Use the existing CheXpert Plus EDA as the starting point for the selected CheXpert data release
- [ ] Build study-level records: `patient_id`, `study_id`, `image_paths[]`, `view_types[]`, `findings`, `impression`, `study_date`, `prior_study_ids[]`
- [ ] Apply patient-level 80/10/10 train/val/test split — strictly no patient leakage across splits
- [ ] De-id normalization: replace `___` with `[DEID]` in all text
- [ ] Remove exact duplicate reports (4 confirmed in CheXpert Plus)
- [ ] Handle sparsity: if `section_findings` absent → use `section_impression` as text target
- [ ] Save clean `dataset_train.csv`, `dataset_val.csv`, `dataset_test.csv`

#### Phase 2: Image Preprocessing (Days 2–4, overlap)
- [ ] Resize to 224x224 (or 512x512 per encoder requirement)
- [ ] Normalize pixel intensities per encoder spec
- [ ] Build PyTorch `ChestXRayDataset` class
- [ ] Tag each image: `frontal` vs `lateral`, `AP` vs `PA`
- [ ] Validate loading success rate; log failures

#### Phase 3: Medical Vision Encoder (Days 4–5)
- [ ] Select encoder: **BioViL-T** (preferred — supports temporal) or **MedCLIP** (simpler fallback)
- [ ] Load pretrained weights from HuggingFace
- [ ] Batch inference → extract embeddings, save as `.npy` / HDF5
- [ ] If multiple views in one study: keep separately before fusion step

#### Phase 4: Contrastive Learning Training (Days 5–7)
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

#### Phase 1: Medical Text Encoder (Days 1–2)
- [ ] Select encoder: **BioLinkBERT-large** or **ClinicalBERT** or **RadBERT**
- [ ] Encode `section_impression` (fallback: `section_findings`) for all training records
- [ ] Batch encode → save embeddings to disk
- [ ] Verify 246-token average aligns with encoder's max length (512)

#### Phase 2: Spatial-Temporal Fusion (Days 4–6, coordinate with A)
- [ ] View fusion: concat frontal + lateral embeddings → project to shared dim
- [ ] Graceful fallback if only frontal available
- [ ] Temporal context: use patient/study identifiers and study dates only when valid longitudinal links are present in the selected CheXpert release
- [ ] Treat any valid prior-study embedding as auxiliary context, never as the primary evidence source
- [ ] If no valid longitudinal links are available, disable the temporal branch and report this as a dataset limitation

#### Phase 3: FAISS Vector DB (Days 3–5)
- [ ] Install `faiss-cpu` (or `faiss-gpu` if GPU available)
- [ ] Index joint embeddings (post-contrastive, coordinate with A)
- [ ] Metadata per indexed item: `study_id`, `patient_id`, `split`, `impression`, `findings`, `pathology_labels`, `view_type`, `study_date`
- [ ] Enforce: test partition NEVER indexed — strict retrieval leakage prevention
- [ ] Implement top-K similarity search function
- [ ] Evaluate retrieval independently: Recall@K, Precision@K, MRR on val set

#### Phase 4: RAG Prompt + LLM (Days 5–7)
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

#### Phase 1: EDA Review & Dataset Documentation (Days 1–2)
- [ ] Review all existing EDA notebooks (`chexpert_plus_core_table_eda.ipynb`, etc.)
- [ ] Extract CheXpert statistics for slides: study/image counts, patient counts, 14 pathology labels, view distribution, report coverage and class imbalance
- [ ] Document the dataset decision: CheXpert is the primary dataset; the report-bearing CheXpert Plus release supplies paired text where required
- [ ] Finalize label strategy: `report.csv` for recall, `impression.csv` for precision

#### Phase 2: Evaluation Framework (Days 2–6)
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

#### Phase 3: Baseline & Ablation Experiments (Days 5–8)
Run on the same val/test set, same protocol:

| Experiment | Config | Expected Metric |
|---|---|---|
| Baseline A | Vision encoder → LLM (no retrieval) | Lowest BLEU/clinical scores |
| Baseline B | Independent embeddings + retrieval | Better retrieval, mid scores |
| Experiment C | Joint contrastive + retrieval | Improved Recall@K |
| Experiment D | C + RAG | Improved BLEU/CheXbert F1 |
| Experiment E | D + multi-view fusion | Marginal improvement |
| Experiment F | E + temporal context, only where valid longitudinal links exist | Evaluate interval-change cases only |
| **Full Model** | All components | Best clinical + factuality scores |

#### Phase 4: Presentation & Report (Days 8–9)
- [ ] Compile all metric results into comparison table
- [ ] Build architecture diagram (2 pipelines: KB construction + RAG generation)
- [ ] Prepare live demo: pick 3–5 test X-rays → show retrieved cases + generated report
- [ ] Write report sections: Introduction, Methodology, Results, Discussion, Conclusion
- [ ] Prepare presentation slides (10–15 slides)

**Deliverable by Oct 10:** Full evaluation table + demo + slides

---

## 📅 Revised Day-by-Day Gantt (Oct 2 → Oct 10)

| Day | Date | Person A | Person B | Person C |
|---|---|---|---|---|
| 1 | Oct 2 | Freeze CheXpert data version; build study/image table | Text encoder setup; test 100 report records | Finalize EDA + 14-label distribution |
| 2 | Oct 3 | Patient-level 80/10/10 split + leakage check | Encode impressions + build text embedding store | Retrieval/evaluation scaffold |
| 3 | Oct 4 | Image preprocessing + PyTorch Dataset | FAISS setup + baseline retrieval | BLEU + ROUGE-L + BERTScore |
| 4 | Oct 5 | BioViL-T/MedCLIP embedding extraction | Top-K retrieval + metadata pipeline | CheXbert evaluation setup |
| 5 | Oct 6 | Contrastive training + hard negatives | RAG prompt + LLM integration | Baseline A/B evaluation |
| 6 | Oct 7 | Contrastive checkpoint evaluation | End-to-end RAG pipeline | Experiments C/D + factuality checks |
| 7 | Oct 8 | Multi-view fusion + final embedding handoff | FAISS re-index + generation quality checks | Experiments E/F where applicable + demo cases |
| 8 | Oct 9 | Final checkpoint + bug fixes | Final RAG pipeline + reproducibility run | Final metrics, comparison table, slides |
| 9 | Oct 10 | **PRESENTATION** | **PRESENTATION** | **PRESENTATION** |

---

## 🔧 Recommended Tech Stack

| Component | Recommended Tool |
|---|---|
| Primary Dataset | CheXpert / CheXpert Plus (report-bearing release) |
| Secondary Dataset | None initially; optional external validation later |
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
| 1 | Clean CheXpert dataset with patient-level splits | ✅ Must have |
| 2 | Image embeddings from pretrained BioViL-T/MedCLIP | ✅ Must have |
| 3 | Text embeddings from BioLinkBERT on impressions | ✅ Must have |
| 4 | FAISS retrieval: top-3 similar cases for a test X-ray | ✅ Must have |
| 5 | LLM generates Findings + Impression from retrieved context | ✅ Must have |
| 6 | BLEU + ROUGE + CheXbert F1 numbers vs. reference | ✅ Must have |
| 7 | Contrastive fine-tuning (Recall@K improvement) | 🔄 Show progress / preliminary numbers |
| 8 | Multi-view fusion ablation; temporal fusion only if valid longitudinal links exist | 🔄 Stretch goal |

> [!TIP]
> For the demo: pick 3 test X-rays with different pathology profiles (e.g., Pleural Effusion, Pneumothorax, No Finding). Show: (1) the image, (2) top-3 retrieved impressions, (3) the LLM-generated report, (4) the reference report, (5) the metric scores.

---

## ⚠️ Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Dataset access / storage | Keep CheXpert as the single primary dataset; begin with the already-EDA'd report-bearing CheXpert release |
| Image download too slow / large | Start with a controlled CheXpert subset, preferably frontal-only for the first retrieval baseline, then add lateral views |
| Contrastive training takes > 4 days | Present pretrained-encoder-only retrieval as Baseline B; show improvement trend |
| LLM API costs | Use Mistral-7B locally via Ollama (free, runs on 8GB VRAM) |
| Report sections sparse | Use `section_impression` as the primary text target; use `section_findings` when available and explicitly record the fallback rate |
| Patient leakage | Create patient-level train/val/test splits and never index validation/test records into the training retrieval store |
| CheXbert setup complexity | Pre-compute CheXbert labels offline; load pre-extracted label files for eval |
| RadGraph setup too complex for timeline | Skip RadGraph F1; focus on CheXbert F1 + BERTScore as clinical proxy |

---

## 🚀 Immediate Next Steps (Today, Oct 2)

> [!IMPORTANT]
> These must be started today.

1. **All 3:** Freeze CheXpert data version, create shared GitHub repo + Google Drive folder for data, checkpoints, embeddings and results
2. **Person A:** Build the CheXpert study/image table and patient-level train/val/test split; start with frontal images
3. **Person B:** `pip install transformers faiss-cpu sentence-transformers sacrebleu rouge-score bert-score`; test text encoding on 100 CheXpert report records
4. **Person C:** Finalize CheXpert EDA statistics, pathology distribution and report-coverage summary; set up CheXbert evaluation

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

*Plan updated: 2026-10-02 — primary dataset changed to CheXpert*
*Sources: `synopsis_report.docx`, `Complete_STREAM_Multimodal_RAG_Methodology.docx`, `chexpert_plus_eda_report.md`, `chexpert_plus_repositories.md`, `df_chexpert_plus_240401_metadata.json`*
