# Exploratory machine-learning ranking of T2D-associated pancreatic β-cell genes

Code, data and outputs for the manuscript:

> **An Exploratory Machine-Learning Ranking of Type 2 Diabetes-Associated Pancreatic β-Cell Genes Using Heterogeneous Gene-Level Evidence**
> Mayishat Altab Mridu, Department of Computer Science and Engineering, Daffodil International University, Dhaka, Bangladesh
> (submitted to *SN Computer Science*)

The study ranks 3,868 genes against a pancreatic β-cell type 2 diabetes (T2D) differential-expression phenotype, using 14 gene-level predictors that span genetic association, expression, tissue specificity, protein–protein interaction and pathway evidence. It then measures how much of the ranking signal comes from gene expression level and detectability rather than from genetic or network evidence.

> **Scope.** The ranking is an exploratory, expression-driven resource for follow-up. It is not a validated multi-evidence predictor, not a clinical tool, and not evidence that any gene is causal for T2D.

---

## Repository contents

| File | Description |
|---|---|
| `T2D_GeneRanking_Analysis_2026_10.ipynb` | Complete analysis notebook (Google Colab), saved with the outputs of the run reported in the manuscript |
| `T2D_GeneRanking_Dataset_3868genes_2026-10.xlsx` | Integrated gene-level dataset used as the notebook's input |
| `T2D_GeneRanking_Outputs_2026-10.zip` | All outputs of that run: tables, figures, workbooks, fitted models and the run manifest |

### Dataset workbook

| Sheet | Contents |
|---|---|
| `FINAL_3868_GENE_DATASET` | One row per gene (3,868 rows): labels, the 14 predictors, grouping variable, source fields and HGNC / Ensembl GRCh38 identifiers |
| `ID_MAPPING_SUMMARY` | Summary of the identifier mapping |
| `DUPLICATE_GENES` | 14 genes recorded under both an old and a current symbol; the redundant unlabeled entries are excluded from candidate lists only |
| `LITERATURE_ANNOTATION` | Hand-compiled, limited literature check for selected candidate genes |

Key columns in `FINAL_3868_GENE_DATASET`:

- `label`: 1 = positive (51 genes), 0 = strict control (1,991), empty = unlabeled (1,826)
- 14 predictors: `in_gwas_evidence`, `in_pisa_evidence`, `in_gse164416_tested`, `ppi_degree`, `gtex_pancreas_tpm`, `gtex_tau_specificity`, `in_kegg_t2d_pathway`, `in_reactome_t2d_pathway`, `gse164416_baseMean`, `gse164416_log2FC`, `lfcSE`, `stat`, `gse164416_pvalue`, `gse164416_FDR`
- `locus_group_v6`: lead-variant grouping proxy used for grouped cross-validation
- `hgnc_symbol`, `hgnc_id`, `ensembl_gene_id_grch38`, `gene_biotype`, `id_mapping_method`: identifiers added after the evidence matrix was assembled
- `gse221156_*`, `emtab5061_*`: label-source and same-cohort fields kept for traceability; **never used as predictors** (checked by an assertion in the notebook)

---

## Data sources

| Source | Role |
|---|---|
| GEO **GSE221156** (β-cell single-cell RNA-seq; Bandesh et al., EMBO J 2026) | Defines the labels only; excluded from all predictors |
| GEO **GSE164416** (islet RNA-seq; Wigger et al., Nat Metab 2021) | Seven expression predictors; supplementary stress test (not independent validation) |
| T2DGGI GWAS (Suzuki et al., Nature 2024) | Genetic-evidence flag and lead-variant grouping proxy |
| PISA islet eQTL (eQTL Catalogue study QTS000033) | Islet regulatory-evidence flag |
| GTEx | Pancreatic expression (TPM) and tissue specificity (tau) |
| STRING | Protein–protein interaction degree |
| KEGG / Reactome | T2D-related pathway membership |
| HGNC complete set (15 Oct 2026); Ensembl BioMart GRCh38.p14 (22 Oct 2026) | Identifier harmonization |

**Label definition (GSE221156 β-cell differential expression):**
- Positive: FDR < 0.05
- Strict control: FDR ≥ 0.05 and |log₂FC| < log₂(1.2)
- Unlabeled: neither definition. These genes are not treated as negatives.

---

## Methods in brief

- **Primary model:** L2-regularized logistic regression with C = 0.1 fixed in advance, median imputation and standardization fitted inside each training fold.
- **Evaluation:** grouped five-fold cross-validation (`StratifiedGroupKFold`, seed 20260910) using the lead-variant grouping proxy. Also:
  - ten repeated grouped splits (seeds 11–110)
  - a fixed-configuration 5 × 3 nested audit
  - 5,000-replicate bootstrap confidence intervals and 3,000-permutation tests
  - held-out Precision@K
- **Secondary benchmark:** nine further classifiers on identical folds. Gradient boosting was the best secondary model; it was not promoted to primary.
- **Diagnostics:**
  - feature-family ablation and single-feature baselines
  - alternative negative sets (all non-significant, near-miss and expression-matched controls)
  - expression-only models and expression-stratified AUC
  - label-definition sensitivity

---

## Key results (reported in the manuscript)

| Quantity | Value |
|---|---|
| Primary ROC-AUC / PR-AUC (single grouped split) | 0.7848 / 0.0749 (prevalence 0.025) |
| Ten repeated splits (mean ± SD) | ROC-AUC 0.771 ± 0.017; PR-AUC 0.072 ± 0.008 |
| 95% bootstrap CI | ROC-AUC 0.7275–0.8387; PR-AUC 0.0507–0.1198 |
| Held-out Precision@10 / 20 / 40 / 100 | 0.10 / 0.05 / 0.10 / 0.09 |
| ROC-AUC vs near-miss / expression-matched negatives | 0.586 / 0.574 |
| GSE164416 supplementary stress test | ROC-AUC 0.5796; PR-AUC 0.2253 |

Most of the discrimination reflects gene expression level and detectability. Genetic-evidence and network/pathway predictors alone do not discriminate.

---

## How to reproduce

1. Open `T2D_GeneRanking_Analysis_2026_10.ipynb` in [Google Colab](https://colab.research.google.com/).
2. Upload `T2D_GeneRanking_Dataset_3868genes_2026-10.xlsx` to `/content`, or let the second cell open the upload dialog. Keep only one workbook in `/content`. The notebook uses whichever `.xlsx` contains the sheet `FINAL_3868_GENE_DATASET`.
3. Choose **Runtime → Run all**. If the first cell repairs the scientific Python stack and restarts the runtime, choose **Run all** again.
4. The last cell packages every output into `T2D_v10_FULL_ML_TRAINING_OUTPUTS.zip` and downloads it.

### Software versions

The reported numbers were produced with Python 3.13, **scikit-learn 1.6.1**, **pandas 2.2.3**, NumPy 2.1.3, SciPy 1.16.3 and matplotlib 3.10. `StratifiedGroupKFold` fold assignment can differ between scikit-learn releases. With other versions, fold-level and pooled metrics may shift slightly; the notebook prints a warning when the versions differ.

### Built-in checks

The notebook stops with an error if any of these fails:
- the gene table does not have 3,868 rows, or the label counts are not 51 / 1,991 / 1,826
- a label-source (GSE221156), E-MTAB-5061 or same-cohort field enters the predictor list
- the held-out predictions are incomplete, or a group appears in both training and test folds
- the nested audit does not reproduce the primary out-of-fold predictions exactly

---

## Outputs (`T2D_GeneRanking_Outputs_2026-10.zip`)

| Path | Contents |
|---|---|
| `run_manifest.json` | Software versions, dataset SHA-256, all headline metrics and audit results |
| `T2D_v10_FULL_ML_FINAL_MODEL_BENCHMARK_RESULTS.xlsx` | Primary metrics, folds, model benchmark, bootstrap, permutation, coefficients, Precision@K |
| `T2D_ADDITIONAL_ANALYSES_FOR_REVISION.xlsx` | Label audit, ten-seed results, paired bootstrap, ablation, label sensitivity, candidate tables |
| `T2D_CANDIDATE_FILTER_ANALYSIS.xlsx` | Candidate lists restricted post hoc to well-observed / expressed genes |
| `T2D_CONFOUND_DIAGNOSTIC.xlsx` | Expression / detectability diagnostics |
| `tables/` | All tables as CSV, including out-of-fold predictions for every model |
| `figures/` | Exploratory and diagnostic figures (PNG and SVG) |
| `models/` | Primary and secondary models fitted on all labeled genes (`joblib`) |

---

## Limitations

- Only 51 positive genes; estimates (especially Precision@K) are imprecise.
- Labels come from differential expression and are not evidence of causality.
- The GSE164416 analysis is not independent validation: GSE164416 also supplies predictors.
- The grouping variable is a lead-variant proxy, not a physical-locus or LD definition.
- Identifiers were harmonized after cross-resource matching, so some genes recorded only by Ensembl ID may have missing predictors.
- Highly ranked unlabeled genes are enriched for poorly observed, median-imputed genes. The well-observed candidate list is a post hoc, descriptive filter.

---

## Contact

Mayishat Altab Mridu, altabmayishat@gmail.com · ORCID [0009-0006-5069-3097](https://orcid.org/0009-0006-5069-3097)

If you use this material, please cite the manuscript once it is published.
