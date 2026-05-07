# ovarian-cancer-regulatory-analysis

# Regulatory Heterogeneity in Ovarian Cancer

A computational exploration of patient subgroups and gene co-expression structure in TCGA-OV data, built as a learning portfolio project to explore cancer omics analysis.

---

## About this project

I built this project to deepen my understanding of cancer genomics by working hands-on with a real ovarian cancer dataset. My background is computer science, machine learning, artificial intelligence, data analysis, data enngineering, neuroAI, biomedical data analysis, so I approached this as someone bringing computational skills to a new biological domain. The goal was not to produce novel research findings, but to demonstrate a complete analytical workflow and to honestly engage with what bulk gene expression data can and cannot reveal.

---

## Dataset

**TCGA-OV PanCancer Atlas** (Ovarian Serous Cystadenocarcinoma) from cBioPortal:

- 300 patients with matched RNA-seq expression and clinical data
- 19,062 genes after cleaning
- Multiple survival endpoints (OS, DSS, PFS, DFS)

Source: https://www.cbioportal.org/study/summary?id=ov_tcga_pan_can_atlas_2018

Note: This particular release of the dataset does not include tumor stage or pre-defined molecular subtypes, which shaped my analytical strategy (see Notebook 02).

---

## Project structure

```
ovarian-cancer-regulatory-analysis/
├── notebooks/
│   ├── 01_data_exploration.ipynb       # Loading, QC, cleaning
│   ├── 02_patient_clustering.ipynb     # PCA, UMAP, K-means, survival
│   └── 03_coexpression_network.ipynb   # Network construction, modules, survival
├── data/
│   ├── raw/                            # Raw cBioPortal data (only used file from the main repo)
│   └── processed/                      # Cleaned outputs from each notebook
├── figures/                            # Saved visualizations
├── pyproject.toml                      # Dependency declarations
├── uv.lock                             # Exact dependency versions
└── README.md
```

---

## Notebook 01: Data exploration and cleaning

**Goal:** Understand the dataset, identify quality issues, and prepare clean data for analysis.

**What I did:**

- Loaded clinical and expression data from cBioPortal
- Explored available clinical variables (survival, demographics, treatment history)
- Found that this dataset version lacks tumor stage and molecular subtype labels
- Identified and resolved several data quality issues:
  - 13 genes with missing names (removed)
  - 19 duplicate gene symbols (kept higher-expression entry)
  - 1,455 genes with no expression values (removed)
  - Small negative values from RSEM floating-point noise (clipped to zero)
- Visualized the expression distribution and observed the characteristic bimodal pattern after log2 transformation, reflecting the on/off nature of gene expression

**Output:** Cleaned dataset of 19,062 genes × 300 patients, with all 300 patients having matched clinical data.

---

## Notebook 02: Patient stratification by clustering

**Goal:** Discover patient subgroups from gene expression data, since the dataset did not provide molecular subtype labels.

**What I did:**

- Selected the top 2,000 most variable genes (after log2 transformation) as the most informative features
- Standardized the data and applied PCA, retaining 30 components (capturing approximately 56% of total variance)
- Used UMAP for 2D visualization and K-means for clustering, testing K from 2 to 8
- Used silhouette scores and the elbow method to evaluate cluster quality

**Key findings:**

- Silhouette scores were uniformly low (maximum 0.137 at K=2), and the UMAP showed one largely continuous cloud of patients rather than distinct groups. This is consistent with what I read in the literature about HGSC: it has been described more as a continuum of molecular states than as a set of sharply defined subtypes.
- I examined K=2 (highest silhouette) and K=4 (commonly reported in the HGSC literature, e.g., the TCGA 2011 Nature paper).
- For K=4, one cluster (Cluster 2, n=92) showed visibly better survival.
- A pairwise log-rank test of Cluster 2 vs. all other clusters combined was statistically significant (p = 0.028; median OS 51 months vs. 35–44 months in other clusters).

**Differential expression analysis** between Cluster 2 and the other clusters revealed that Cluster 2 is characterized by elevated expression of immune-related genes including GZMB, GNLY, CXCL9, CXCL10, CXCL11, CXCL13, TAP1, TAP2, IRF1, and MS4A1. After looking up these genes, I learned this is consistent with what the literature describes as the "immunoreactive" subtype of high-grade serous ovarian cancer which is a known subgroup with relatively better prognosis. I did not know this before running the analysis.

---

## Notebook 03: Gene co-expression network analysis

**Goal:** Identify groups of genes that vary together across patients, and ask whether these groups correspond to coherent biological functions and clinical outcomes.

**What I did:**

- Computed Spearman correlations between the top 1,000 most variable genes (~500,000 pairs)
- Built an undirected network with edge threshold |r| > 0.6
- Identified communities using the Louvain algorithm
- Computed per-patient module activity scores (mean z-scored expression of module genes) and tested association with survival via median split + log-rank test

**Key findings:**

- The network had 132 connected genes, 1,618 edges, and 6 communities.
- The top five modules were biologically interpretable based on their hub genes:

| Module       | Theme                                         | Top hubs                           |
| ------------ | --------------------------------- | ---------------------------------- |
| 1 (52 genes) | Stromal / cancer-associated       | LUM, FAP, SFRP2, POSTN, PRRX1      |
| 2 (36 genes) | Immune                            | CXCL9, IL21R, GZMK, PLA2G2D, IRF4  |
| 3 (30 genes) | Ovarian developmental identity    | COLEC11, STAR, ARX, FOXL2, C3orf72 |
| 4 (7 genes)  | Acute-phase inflammation          | NNMT, LCN2, SAA1, SAA2, FMO2       |
| 5 (4 genes)  | Adipose / omentum                 | FABP4, CCL21, ADH1B, ADIPOQ        |

- Two modules showed significant or trending survival associations:
  - **Immune module:** patients with high module score showed a trend toward better survival (p = 0.056)
  - **Acute-phase module:** patients with high module score had significantly worse survival (p = 0.018)

---

## Two analyses, similar conclusions

The clustering analysis (Notebook 02) and the network analysis (Notebook 03) were performed independently but pointed to overlapping biology around the immune signature. The genes upregulated in the survival-favorable Cluster 2 substantially overlap with the genes in the immune network module. I found this convergence reassuring as a basic sanity check, although it is not independent evidence in a strict statistical sense, both analyses use the same expression matrix.

---

## Limitations

I want to be upfront about what this project does and does not show:

- **I am new to cancer biology.** My background is computer science and machine learning. I learned much of the relevant biology while doing this project, including the names and roles of specific genes and the proposed molecular subtypes of HGSC. Some interpretations may be naive or imprecise.
- **The statistical methods are basic.** I used log-rank tests with median splits rather than Cox proportional hazards regression, which would be more rigorous.
- **No multiple testing correction** was applied to differential expression p-values, so individual gene-level significance should be treated as exploratory.
- **Co-expression is not regulation.** Two correlated genes are not necessarily in a regulatory relationship. Methods like PANDA integrate transcription factor binding motifs and protein-protein interactions to construct true regulatory networks, and methods like LIONESS extend this to per-patient networks (as per my understanding, I might be wrong). My co-expression analysis is a simplified starting point relative to these.
- **Sample size (n = 300) limits statistical power.** Borderline results (e.g., immune module p = 0.056) likely reflect underpowered detection rather than a true absence of effect.

---

## What I would do next

If extending this project, the natural directions would be:

- **Learn Cox regression properly** to redo the survival analyses with appropriate confounder adjustment.
- **Apply the Kuijjer lab's tools** — PANDA for population-level regulatory networks, LIONESS for sample-specific regulatory networks.
- **Integrate single-cell RNA-seq data** to dissect tumor-intrinsic vs. microenvironment contributions to the modules.
- **Validate findings in independent cohorts** (e.g., GSE9891) to test robustness.
- **Integrate mutation and copy-number data** that are also available in the cBioPortal dataset.

---

## Reproducing this analysis

### Data download

Download the TCGA-OV PanCancer Atlas dataset from cBioPortal:
https://www.cbioportal.org/study/summary?id=ov_tcga_pan_can_atlas_2018

Place the unzipped files in the `data/` folder.

### Environment setup

This project uses [uv](https://docs.astral.sh/uv/) for dependency management.

```bash
git clone <repo-url>
cd ovarian-cancer-regulatory-analysis

# Install exact dependency versions from uv.lock
uv sync

# Activate the environment
source .venv/bin/activate

# Launch Jupyter
jupyter notebook
```

Run notebooks in order: `01_data_exploration.ipynb`, `02_patient_clustering.ipynb`, `03_coexpression_network.ipynb`.

---

## Tools and methods

| Purpose                  | Library                                |
| ------------------------ | -------------------------------------- |
| Data manipulation        | pandas, numpy                          |
| Visualization            | matplotlib, seaborn                    |
| Dimensionality reduction | scikit-learn (PCA), umap-learn         |
| Clustering               | scikit-learn (K-means)                 |
| Differential expression  | scipy.stats (Welch's t-test)           |
| Network analysis         | networkx (Louvain community detection) |
| Survival analysis        | lifelines (Kaplan-Meier, log-rank)     |

---

## Acknowledgments

Data accessed from The Cancer Genome Atlas (TCGA) via cBioPortal.

This project was built as a learning exercise. I used Claude (an AI assistant) to help me understand cancer biology concepts I was unfamiliar with, and debug some code. The choice of analyses, the framing of findings, the limitations I have flagged, and the decision to keep the project focused on methods I genuinely understand are my own.
