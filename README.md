# T2D v10: Gene Prioritization ML Training Notebook

`T2D_Dataset_Training.ipynb` trains and evaluates a machine-learning model that prioritizes genes for a **beta-cell type 2 diabetes (T2D) differential-expression phenotype**. It runs on Google Colab and uses the finalized v10 dataset workbook.

The model predicts the defined beta-cell T2D DE phenotype. It does **not** establish causality.


## Overview

| Item | Value |
|---|---|
| Task | Binary classification of genes (positive beta-cell T2D DE vs. strict control) |
| Primary model | L2 logistic regression, **C = 0.1** (locked, no post-hoc tuning) |
| Features | 14 locked predictors |
| Evaluation | Grouped 5-fold out-of-fold (OOF) CV, with a 5×3 nested protocol audit |
| Main metrics | ROC-AUC, PR-AUC, Brier score, log loss, Precision@K |
| Seed | `20260910` |
| Environment | Python 3.13, NumPy, SciPy, pandas, scikit-learn 1.6.1 |

---

## Dataset

**Input file:** `T2D.xlsx`

The input file is not included here, so upload it when the notebook prompts for it. The notebook prints its SHA-256 hash for traceability.

**Sheet used:** `FINAL_3868_GENE_DATASET` (3,868 genes)

| Group | Genes |
|---|---|
| Positive (`label` = 1) | 51 |
| Strict controls (`label` = 0) | 1,991 |
| Unlabeled (NaN, excluded from training) | 1,826 |
| **Labeled total** | **2,042** (prevalence ≈ 2.5%) |

**Grouping:** `locus_group_v6` (1,951 unique groups). Genes with a missing group ID become singleton groups, and no locus is invented. Groups keep genes from the same locus together in the same CV fold.

### The 14 locked predictors

| Category | Features |
|---|---|
| Genetic / literature evidence | `in_gwas_evidence`, `in_pisa_evidence` |
| Pathway membership | `in_kegg_t2d_pathway`, `in_reactome_t2d_pathway` |
| Network | `ppi_degree` |
| Tissue expression (GTEx) | `gtex_pancreas_tpm`, `gtex_tau_specificity` |
| GSE164416 differential expression | `in_gse164416_tested`, `gse164416_baseMean`, `gse164416_log2FC`, `lfcSE`, `stat`, `gse164416_pvalue`, `gse164416_FDR` |

### Leakage controls

- Any column starting with `gse221156_` or `emtab5061_` is **forbidden** as a predictor. These are the label-source and related datasets.
- No same-cohort "2026" beta-cell fields are used.
- Hard assertions in the notebook fail the run if a forbidden feature enters the allow-list.
- Imputation (median, no indicator columns) and scaling are fitted **inside each training fold** via sklearn pipelines.

---

## Method

1. **Environment check.** The first cell tests the NumPy/SciPy/pandas/sklearn binaries. If they are mismatched, it reinstalls a pinned stack and restarts the kernel.
2. **Data load and audit.** It loads the gene sheet, checks the 14 features, runs the leakage audit, and builds `X`, `y`, and `groups`.
3. **Primary evaluation.** The fixed L2 LR is evaluated with `StratifiedGroupKFold` (5 outer folds). A 5×3 nested grouped structure is also run as a protocol audit, and **no hyperparameter is tuned**.
4. **Robustness.** Repeated grouped 5-fold CV runs over 5 seeds (11, 22, 33, 44, 55).
5. **Secondary benchmark.** Ten models are compared on identical grouped folds: L2 LR (primary), ElasticNet LR, Random Forest, Extra Trees, HistGradientBoosting, Gradient Boosting, RBF SVM, KNN, Gaussian NB, and shrinkage LDA. The benchmark is exploratory, and **no secondary model replaces the primary**.
6. **Statistics.** It computes bootstrap 95% CIs (5,000 replicates) and OOF label-permutation tests (3,000 permutations).
7. **Interpretation.** It reports LR coefficients and fold-wise permutation feature stability.
8. **Gene ranking.** The primary model is fit on all labeled genes and scores all 3,868 genes. It reports Precision@K and enrichment, plus an exploratory secondary-model ranking.
9. **Supplementary GSE164416 evaluation.** It uses only the non-GSE164416 features.
10. **Reproducibility.** It writes a run manifest and a QA assertion table, then packages everything into a ZIP.

---

## Results

Metrics are grouped out-of-fold results, taken from the notebook's saved outputs.

### Primary model (L2 LR, C = 0.1)

| Metric | Value | 95% bootstrap CI |
|---|---|---|
| ROC-AUC | 0.7848 | 0.727–0.839 |
| PR-AUC (average precision) | 0.0749 | 0.051–0.120 |
| Brier score | 0.0242 | – |
| Log loss | 0.1124 | – |

Permutation-test p-values are 0.00033 (ROC-AUC) and 0.00067 (PR-AUC). The prevalence baseline for PR-AUC is 0.025.

### Model benchmark (ranked by PR-AUC)

| Model | ROC-AUC | PR-AUC |
|---|---|---|
| Gradient Boosting | 0.8001 | 0.1835 |
| Random Forest | 0.7627 | 0.1333 |
| KNN | 0.6564 | 0.0884 |
| HistGradientBoosting | 0.7186 | 0.0879 |
| Extra Trees | 0.7147 | 0.0874 |
| **L2 LR (locked primary)** | **0.7848** | **0.0749** |
| Shrinkage LDA | 0.7820 | 0.0731 |
| ElasticNet LR | 0.7781 | 0.0695 |
| Gaussian NB | 0.6710 | 0.0615 |
| RBF SVM | 0.7096 | 0.0534 |

Gradient Boosting scores highest on PR-AUC, but the primary model was locked in advance and stays primary. Gradient Boosting is reported only as a secondary, exploratory benchmark.

### Precision@K (primary model, labeled genes)

| K | Hits | Precision | Enrichment |
|---|---|---|---|
| 10 | 2 | 0.200 | 8.0× |
| 20 | 4 | 0.200 | 8.0× |
| 40 | 5 | 0.125 | 5.0× |
| 50 | 8 | 0.160 | 6.4× |
| 100 | 14 | 0.140 | 5.6× |
| 200 | 22 | 0.110 | 4.4× |
| 500 | 38 | 0.076 | 3.0× |

---

## How to run

1. Open the notebook in **Google Colab**.
2. Run **Cell 1** (environment check). If it reports a broken stack, it repairs the packages and **restarts the kernel**. After the restart, choose **Runtime → Run all** again. Do not add `pip install -U numpy/scipy/pandas/scikit-learn` anywhere, because mixing upgraded binaries caused the original `_slice` import error.
3. When prompted (Cell "2. Upload dataset"), upload `T2D_IEEE_FINAL_SUBMISSION_DATASET_v10_FINALIZED.xlsx`.
4. Run the remaining cells in order. The final cells zip and download the outputs.

**Dependencies (pinned):** numpy 2.2.6, scipy 1.15.3, pandas 2.2.3, scikit-learn 1.6.1, matplotlib 3.10.1, seaborn 0.13.2, joblib 1.4.2, openpyxl 3.1.5.

---




Key gene-ranking files:
- `top100_gene_prioritization.csv`: top 100 genes by primary-model score.
- `all_gene_scores.csv`: score and rank for all 3,868 genes.
- `top100_best_secondary_model_prioritization.csv`: exploratory Gradient Boosting ranking.

---


---

