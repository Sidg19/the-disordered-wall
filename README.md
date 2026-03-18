# The Disordered Wall

## Summary

This project asks whether mutation effects behave differently in intrinsically disordered regions (IDRs) than in ordered protein regions, and whether the same predictive cues work equally well in both settings. To study this, I built a benchmark by intersecting ProteinGym deep mutational scanning (DMS) assays with DisProt disorder annotations, labeling each mutation as either `IDR` or `ordered`, and comparing both mutation-effect distributions and predictive model performance across region types.

In the final benchmark, mutations in IDRs were slightly more tolerated on average than mutations in ordered regions at the pooled dataset level, although this effect was heterogeneous across proteins. Predictive performance was consistently lower in IDRs than in ordered regions. As a bonus extension, I added local sequence-context features around each mutation and found that these features substantially improved mutation-effect prediction beyond mutation identity and a coarse IDR/ordered label.

---

## Biological Problem and Motivation

Intrinsically disordered regions are biologically important but harder to describe using standard structure-centric tools. Ordered domains are often analyzed in terms of stable 3D structure, residue packing, burial/exposure, and structure-confidence metrics, but many IDRs do not adopt one fixed conformation. That raises two related questions:

1. Do mutation-effect scores follow different distributions in IDRs compared with ordered regions?
2. Are mutation effects harder to model in IDRs, and if so, what kinds of features remain useful?

I was interested in this question because my broader research interests sit at the interface of protein/RNA biology, disorder, and computational modeling. The project also connects to a larger theme in protein machine learning: how far structure-based intuition transfers to flexible, disordered biology.

Although the original route was partly motivated by structure-derived signals such as pLDDT, the final successful analysis ultimately focused on mutation-level, disorder-aware, and sequence-context features rather than structure-derived features.

---

## Public Dataset Links

The raw benchmark files used in this project are public and can be downloaded from the official sources below.

### ProteinGym
- [ProteinGym download page](https://proteingym.org/download)
- [ProteinGym official GitHub repository](https://github.com/OATML-Markslab/ProteinGym)

Files used from ProteinGym:
- `DMS_substitutions.csv`
- `DMS_ProteinGym_substitutions/`
- `ProteinGym_AF2_structures/` *(downloaded during setup, but not part of the final successful analysis)*

### DisProt
- [DisProt download page](https://disprot.org/download)
- [DisProt homepage](https://disprot.org/)

Files used from DisProt:
- `DisProt_release_2025_12.tsv`

### Note
Raw benchmark data are not committed to this repository in order to keep the repo lightweight and because the datasets are already publicly available from the official sources above.

More detailed setup instructions are provided in [`data_docs/DATA_SETUP.md`](data_docs/DATA_SETUP.md).

---

## What Was Used in the Final Analysis

After filtering and overlap construction, the final benchmark contained:

- **34 ProteinGym assays**
- **32 proteins**
- **48,706 variants total**
- **11,813 IDR variants**
- **36,893 ordered variants**

These variants came from the ProteinGym × DisProt intersection after:
- UniProt identifier mapping
- single-mutation parsing
- sequence-position validation
- region labeling

---

## Computational Approach

### Benchmark construction

The main pipeline does the following:

1. Load ProteinGym assay metadata and DMS assay files.
2. Map ProteinGym identifiers to UniProt accessions so they can be intersected with DisProt.
3. Load DisProt disorder annotations.
4. Intersect ProteinGym proteins with DisProt proteins.
5. Parse single amino-acid substitutions (for example, `A25V`).
6. Validate mutation positions against the assay target sequence.
7. Label each mutation as `IDR` or `ordered` based on DisProt residue ranges.
8. Standardize assay scores within each assay to create `DMS_score_z`.

### Feature engineering

The baseline mutation-level feature set included:

- wild-type residue
- mutant residue
- position within sequence
- change in hydropathy
- change in volume
- change in charge
- conservative vs non-conservative substitution indicators
- simple residue class indicators

I then added:
- a coarse `region_type` feature (`IDR` vs `ordered`)
- local disorder density

For the bonus extension, I added local sequence-context features computed from a ±5 residue window around the mutation site, including:

- local charged fraction
- local positive / negative fraction
- local hydrophobic fraction
- local polar fraction
- local aromatic fraction
- glycine/proline fraction
- local net charge
- local mean hydropathy
- a simple low-complexity proxy

### Modeling workflow

The predictive task was:

**Predict assay-standardized mutation effect (`DMS_score_z`) from mutation/site features.**

I used grouped cross-validation by protein (`GroupKFold`) so that evaluation was protein-aware rather than allowing within-protein leakage.

The main models compared were:

1. `mutation_only`
2. `mutation_plus_region`
3. `mutation_plus_region_plus_context` *(bonus extension)*

Performance was evaluated using:

- Spearman correlation
- RMSE
- R²

and was reported overall as well as separately for IDR and ordered subsets.

The core regression model was a **RandomForestRegressor** inside a preprocessing pipeline with one-hot encoding for categorical features and standard preprocessing for numeric features.

---

## Route Design and What Was Completed

### Planned route idea

The original route asked whether the “handholds” that work on ordered protein structure — especially structure-derived confidence signals — become less useful in IDRs, and whether mutation-effect distributions differ in disordered regions.

### Completed successfully

- Built a ProteinGym × DisProt benchmark
- Mapped mutation positions to IDR vs ordered regions
- Compared mutation-effect distributions across region types
- Trained grouped cross-validated predictive baselines
- Evaluated model performance separately for IDR and ordered mutations
- Performed balanced subsampling and protein-level robustness analyses
- Added a bonus extension using local sequence-context features

### Attempted but not completed successfully

- Structure-derived feature integration (especially pLDDT / AlphaFold-derived features)

The final project therefore answers the main comparative and predictive questions, but it does **not** support strong benchmark-wide conclusions about structure-derived features in the final reported model.

---

## Key Results

### 1. Mutation-effect distributions differ between IDRs and ordered regions

In the final intersected benchmark:

- IDR mean `DMS_score_z` ≈ **0.031**
- ordered mean `DMS_score_z` ≈ **-0.010**
- IDR median `DMS_score_z` ≈ **0.274**
- ordered median `DMS_score_z` ≈ **0.181**

This indicates that, on average, mutations in IDRs were **slightly more tolerated** than mutations in ordered regions in the pooled benchmark.

### 2. The pooled IDR-vs-ordered difference is robust to class imbalance

Because the benchmark contains many more ordered than IDR variants, I performed balanced subsampling. The result remained positive:

- balanced mean difference (`IDR - ordered`) ≈ **0.0409 ± 0.0078**
- 95% interval for balanced mean difference ≈ **[0.0276, 0.0567]**
- balanced median difference ≈ **0.0941 ± 0.0084**
- 95% interval for balanced median difference ≈ **[0.0831, 0.1108]**

This suggests the shift toward more tolerated IDR mutations is **not simply caused by the larger number of ordered variants**.

### 3. The protein-level effect is heterogeneous

Among proteins with both region types represented:

- **12 proteins** had both IDR and ordered mutations
- **7** showed positive `IDR - ordered` median differences
- **5** showed negative differences
- Wilcoxon paired p-value ≈ **0.733**

This suggests the pooled IDR effect is **not universal across proteins**. Some proteins show more tolerated IDR mutations, while others show the opposite.

### 4. Mutation-effect prediction is worse in IDRs than in ordered regions

For the baseline predictive models, Spearman correlation was consistently lower in IDRs than in ordered regions.

Approximate mean Spearman values:

- `mutation_only`: IDR ≈ **0.122**, ordered ≈ **0.221**
- `mutation_plus_region`: IDR ≈ **0.110**, ordered ≈ **0.227**

This supports the idea that mutation effects are **harder to rank/predict in IDRs** than in ordered protein regions.

### 5. A coarse IDR/ordered label alone adds very little

Adding only the `region_type` label produced very little overall improvement compared with mutation-only features.

This suggests that a coarse `IDR` / `ordered` label is too simple to capture the sequence determinants that matter for mutation effects.

### 6. Bonus extension: local sequence context substantially improves prediction

I added a third model, `mutation_plus_region_plus_context`, using ±5 residue local sequence-context features.

This improved mean Spearman substantially:

- IDR: from ≈ **0.110** to ≈ **0.144**
- all variants: from ≈ **0.208** to ≈ **0.256**
- ordered: from ≈ **0.227** to ≈ **0.292**

The improvement over `mutation_plus_region` was approximately:

- IDR: **+0.033**
- all: **+0.048**
- ordered: **+0.065**

This bonus result suggests that **local sequence environment carries useful predictive information beyond mutation identity and a coarse region label**.

---

## Interpretation

The final picture is more nuanced than a simple “IDRs are different” claim.

### What the results support

- IDR mutations are somewhat more tolerated on average in the pooled benchmark.
- This pooled difference is robust to class imbalance.
- Mutation-effect prediction is harder in IDRs than in ordered regions.
- A coarse IDR/ordered label is not enough to improve prediction much.
- Local sequence context provides substantial additional signal.

### What the results do **not** support strongly

- that the IDR-vs-ordered effect is universal across proteins
- that structure-derived features were successfully benchmarked in the final analysis
- that local sequence context helps IDRs more than ordered regions specifically

In fact, the context bonus improved prediction in **both** region types, with an even larger gain in ordered regions in this benchmark.

---

## Limitations and Future Work

A key limitation of the current workflow is that structure-derived features were not robustly incorporated into the final benchmark.

Although AlphaFold-style PDB files were available, structure-derived features were not included in the final benchmark because doing so would have required a separate and carefully validated mutation-to-structure mapping pipeline. This was not just a matter of finding the right file names: it would also have required residue-level alignment between ProteinGym assay positions and structure residue numbering, handling partial or split structures, verifying wild-type residue identity, and then post-processing coordinates to extract meaningful geometric features. Since those steps were substantial and a naive implementation risked introducing silent mapping errors, the final project focused on mutation-level, disorder-aware, and local sequence-context features instead.

More importantly, pLDDT alone was not treated as a strong enough feature to justify rebuilding the final model around structure confidence. AlphaFold is already expected to assign low confidence to intrinsically disordered regions, so in this benchmark pLDDT would mainly behave as a coarse disorder/confidence marker rather than a rich mutation-effect descriptor. In other words, it would largely restate that a region is poorly structured, without providing much additional mutation-level information. 

For that reason, the final successful analysis focused on mutation descriptors, disorder annotations, and local sequence-context features, which produced clearer and more interpretable gains.

If I extended the project further, I would prioritize:

1. Repairing structure-file matching so mutation rows can be mapped reliably to available PDB files.
2. Extracting richer structure-derived features from the available PDBs, such as local contact count, packing density, or approximate exposure, rather than relying on pLDDT alone.
3. Expanding the sequence-context model with richer composition or motif-aware features.
4. Running protein-level or assay-level models to better separate global trends from protein-specific effects.
5. Testing whether the same conclusions hold under stricter external validation or additional DMS benchmarks.

---

## Reproducibility / How to Run

### Main notebook

Run the final notebook in Google Colab or Jupyter:

- [`notebooks/disordered_wall_pipeline_clean.ipynb`](notebooks/disordered_wall_pipeline_clean.ipynb)

### Required inputs

Place the following inputs where the notebook expects them:

- ProteinGym DMS substitutions reference CSV
- ProteinGym DMS assay CSV folder
- DisProt regions TSV
- optional ProteinGym AlphaFold structure folder

### Basic workflow

1. Mount Google Drive (if using Colab).
2. Set the input file paths in the configuration cell.
3. Run the notebook sections in order:
   - setup and imports
   - data loading
   - ProteinGym × DisProt intersection
   - region labeling and feature engineering
   - modeling
   - robustness analyses
   - bonus sequence-context extension
4. Export figures/tables as needed.

### Main dependencies

Typical dependencies used in the notebook:

- Python 3
- pandas
- numpy
- matplotlib
- scipy
- scikit-learn
- biopython
- requests

See [`requirements.txt`](requirements.txt) and [`data_docs/DATA_SETUP.md`](data_docs/DATA_SETUP.md) for more details.

---
