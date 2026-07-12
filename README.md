# Single-Cell Aging Clock

This is an undergraduate bioinformatics project where I tried to predict mouse age from single-cell RNA-seq data.

The main idea was to check whether gene expression patterns from individual cells contain enough information to separate young, middle-aged, and old mice.

I used Python, Scanpy, AnnData, scikit-learn, Random Forest models, and SHAP for model interpretation.

This project is still exploratory. It is not meant to be a perfect biological aging clock. I mainly built it to understand single-cell data analysis, machine learning on biological data, and how to interpret model results carefully.

---

## Why I made this project

I wanted to build a project that combines:

* single-cell RNA-seq
* aging biology
* machine learning
* model interpretation

At first, I did not directly start with the aging dataset. I used Scanpy’s PBMC3k dataset to understand how AnnData objects, QC, PCA, UMAP, and clustering work. After that, I moved to the actual aging dataset.

---

## Dataset

The main dataset used in this project is a processed subset of **Tabula Muris Senis**, a mouse aging single-cell dataset.

The dataset contains:

* 5,218 cells
* 22,966 genes
* 3 age groups:

  * 3 months
  * 18 months
  * 24 months

Important metadata columns used:

* `age`
* `age_months`
* `mouse.id`
* `sex`
* `tissue`
* `cell_ontology_class`

The `.h5ad` dataset file is not uploaded to GitHub because it is too large. Locally, I used:

```text
data/processed/tabula_muris_senis_dev_subset.h5ad
```

---

## Repository structure

```text
single-cell-aging-clock/
│
├── notebooks/
│   ├── 01_data_loading_and_QC.ipynb
│   ├── 02_data_analysis.ipynb
│   ├── 03_tabula_muris_subset_preparation.ipynb
│   ├── 04_exploratory_analysis_tabula_muris.ipynb
│   ├── 05_aging_clock_model.ipynb
│   └── 06_gene_level_model_and_shap.ipynb
│
├── data/
│   ├── raw/
│   └── processed/
│
├── results/
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Notebook summary

### 01_data_loading_and_QC.ipynb

This notebook uses the PBMC3k dataset from Scanpy.

I used this as a practice dataset because I was still learning the basic single-cell workflow.

In this notebook, I did:

* data loading
* QC checks
* filtering low-quality cells
* normalization
* log transformation
* highly variable gene selection

PBMC3k does not have age labels, so it is not used for the aging model.

---

### 02_data_analysis.ipynb

This notebook continues with the PBMC3k data.

I used it to practice:

* PCA
* nearest-neighbor graph construction
* UMAP
* Leiden clustering

This helped me understand how cells can be visualized and grouped based on expression patterns.

---

### 03_tabula_muris_subset_preparation.ipynb

This is where I moved from practice data to the actual aging dataset.

In this notebook, I:

* loaded the Tabula Muris Senis subset
* checked the metadata columns
* confirmed that age labels were present
* converted age groups into numeric values

The age mapping was:

```text
3m  -> 3
18m -> 18
24m -> 24
```

This created the `age_months` column, which was used as the prediction target later.

---

### 04_exploratory_analysis_tabula_muris.ipynb

In this notebook, I explored the Tabula Muris Senis data.

The dataset was already processed and already had PCA, UMAP, and clustering information, so I did not treat this notebook as raw preprocessing.

I checked:

* age distribution
* tissue distribution
* cell-type distribution
* UMAP colored by age
* UMAP colored by tissue
* UMAP colored by cell type

This helped me see that the dataset has a lot of biological variation, not just age variation.

---

### 05_aging_clock_model.ipynb

In this notebook, I built the first age-prediction models.

A very important issue was data leakage. Since many cells come from the same mouse, I did not randomly split individual cells. Instead, I split the data by `mouse.id`.

Final split:

```text
Training cells: 3894
Testing cells: 1324
Training mice: 10
Testing mice: 4
Overlapping mice: 0
```

I tested three models:

| Model                                | Cell-level MAE | Cell-level R² |
| ------------------------------------ | -------------: | ------------: |
| Mean-age baseline                    |          ~7.59 |        ~-0.04 |
| Random Forest using precomputed PCA  |          ~6.02 |         ~0.28 |
| Random Forest using train-fitted SVD |          ~6.11 |         ~0.20 |

The PCA model performed a little better, but the PCA values were already present in the AnnData object before my train/test split. Because of that, I treated it only as an exploratory baseline.

The SVD model was cleaner because SVD was fitted only on the training cells and then applied to the test cells.

---

### 06_gene_level_model_and_shap.ipynb

In this notebook, I moved from PCA/SVD features to actual gene-level features.

First, I trained a gene-level Random Forest model using all cell types together.

That model performed badly:

| Model                 | Cell-level MAE | Cell-level R² |
| --------------------- | -------------: | ------------: |
| Mixed-cell gene model |          7.279 |        -0.065 |

This showed that using all cell types together was probably too noisy. The model may have been learning cell type, tissue, sex, or mouse-specific patterns instead of age.

After that, I selected one well-represented cell type: **bronchial smooth muscle cell**.

This model performed better:

| Model                                   | Cell-level MAE | Cell-level R² |
| --------------------------------------- | -------------: | ------------: |
| Bronchial smooth muscle cell gene model |          4.488 |         0.326 |

This suggests that age-related signal was easier to detect after reducing cell-type heterogeneity.

Then I applied SHAP to the bronchial smooth muscle cell model.

Top SHAP genes included:

* `Iapp`
* `Csnk2a1`
* `C1qb`
* `Krt15`
* `Nptn`
* `Tubb4a`
* `Cx3cr1`
* `S100a9`
* `C1qa`
* `C1qc`
* `Notch2`
* `Gja1`

I interpreted these genes carefully. Some immune-related genes like `C1qa`, `C1qb`, `C1qc`, `Cx3cr1`, and `S100a9` may reflect inflammatory or immune-related age signals.

However, some genes like `Iapp`, `Alb`, and `Cel` are not typical bronchial smooth muscle cell markers. These could be due to ambient RNA, annotation noise, doublets, or background tissue signal.

So I do not claim these are confirmed aging biomarkers. They are only genes that were important for this model’s predictions.

---

## Main things I learned

* Single-cell data needs careful handling because cells from the same mouse are not independent.
* Splitting by mouse is better than randomly splitting cells.
* Mixed-cell models can perform badly because different cell types have very different expression patterns.
* Cell-type-specific modeling can reduce some noise.
* SHAP is useful, but it explains the model, not the actual biology directly.
* A model can give gene importance values even when the biology is still uncertain.

---

## Main results

The best model in this project was the bronchial smooth muscle cell gene-level model.

```text
Cell-level MAE: 4.488 months
Cell-level R²: 0.326
```

This is not a perfect result, but it is better than the mixed-cell gene model and the simple baseline models.

The result suggests that cell-type-specific age prediction may be more useful than using all cells together.

---

## Tools used

* Python
* Scanpy
* AnnData
* pandas
* NumPy
* scikit-learn
* SHAP
* matplotlib
* Jupyter Notebook
* Git and GitHub

---

## How to run this project

Clone the repository:

```bash
git clone https://github.com/p5lak/single-cell-aging-clock.git
cd single-cell-aging-clock
```

Create a virtual environment:

```bash
python -m venv aging-clock-env
source aging-clock-env/bin/activate
```

Install requirements:

```bash
pip install -r requirements.txt
```

Place the processed dataset here:

```text
data/processed/tabula_muris_senis_dev_subset.h5ad
```

Start Jupyter:

```bash
jupyter notebook
```

Run the notebooks in order from `01` to `06`.

---

## Limitations

This project has many limitations:

* The dataset has only three age groups.
* The test set has only four mice.
* Cells from the same mouse are related to each other.
* Some models may still be affected by sex, tissue, cell-type, or mouse-specific effects.
* The dataset was already processed, so some preprocessing choices were not controlled by me.
* SHAP genes are not automatically biological aging markers.
* The top genes would need validation using another dataset or experimental evidence.

---

## Future improvements

Some things I would like to improve later:

* use leave-one-mouse-out cross-validation
* test more cell types separately
* compare Random Forest with simpler models like ElasticNet
* do pathway enrichment analysis on SHAP-ranked genes
* validate top genes in another aging dataset
* control more carefully for sex and tissue effects
* clean repeated code into helper scripts

---

## Project status

This is a learning-focused undergraduate project. It helped me understand how single-cell data is processed, how leakage can happen in biological ML, and why model interpretation needs to be done carefully.

