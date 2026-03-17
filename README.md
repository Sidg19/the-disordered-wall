# The Disordered Wall

## Summary
This project asks whether mutation effects behave differently in intrinsically disordered regions (IDRs) versus ordered protein regions, and whether the same predictive cues work equally well in both settings. I built a benchmark by intersecting ProteinGym deep mutational scanning (DMS) assays with DisProt disorder annotations, labeled each mutation as IDR or ordered, and compared both mutation-effect distributions and predictive model performance across region types. In the final benchmark, IDR mutations were slightly more tolerated on average than ordered-region mutations, but predictive performance remained lower in IDRs. As a bonus extension, I added local sequence-context features around each mutation and found that these features substantially improved prediction beyond mutation identity and a coarse IDR/ordered label.

## Biological Problem and Motivation
Intrinsically disordered regions are biologically important but harder to describe using standard structure-centric tools. Ordered domains are often analyzed through stable 3D structure, residue packing, and confidence metrics, but many IDRs do not adopt one fixed conformation. This raises two related questions:

1. Do mutation-effect scores follow different distributions in IDRs compared with ordered regions?
2. Are mutation effects harder to model in IDRs, and if so, what kinds of features are still useful?

I was interested in this question because my broader research interests sit at the interface of protein/RNA biology, disorder, and computational modeling. The project also connects to a bigger theme in protein ML: how far structure-based intuition transfers to flexible, disordered biology.

## Public dataset links

The raw benchmark files used in this project are public and can be downloaded from the official sources below.

### ProteinGym
- [ProteinGym download page](https://proteingym.org/download)
- [ProteinGym official GitHub repository](https://github.com/OATML-Markslab/ProteinGym)

Files used from ProteinGym:
- `DMS_substitutions.csv`
- `DMS_ProteinGym_substitutions/`
- `ProteinGym_AF2_structures/` *(downloaded during setup, but structure-derived features were not part of the final successful analysis)*

### DisProt
- [DisProt annotated proteins download page](https://disprot.org/download)
- [DisProt homepage](https://disprot.org/)

Files used from DisProt:
- `DisProt_release_2025_12.tsv`

### Note
Raw benchmark data are not committed to this repository to keep the repo lightweight and because the datasets are publicly available from the official sources above.
### 2. DisProt
I used DisProt annotated disorder regions to assign each mutation position to either an IDR or an ordered region.

### 3. Protein Structures / pLDDT
I downloaded the ProteinGym AlphaFold structure set with the intention of incorporating structure-derived features such as pLDDT. However, structure coverage did not populate successfully in the final benchmark, so the final reported analyses do **not** rely on structure-derived features. I mention this explicitly because it was part of the original route idea, but it was not a successful component of the final analysis.

## What Was Used in the Final Analysis
After filtering and overlap construction, the final benchmark contained:

- 34 ProteinGym assays
- 32 proteins
- 48,706 variants total
- 11,813 IDR variants
- 36,893 ordered variants

These variants came from the ProteinGym × DisProt intersection after UniProt identifier mapping, single-mutation parsing, sequence-position validation, and region labeling.

## Computational Approach
### Benchmark construction
The pipeline does the following:

1. Load ProteinGym assay metadata and DMS assay files.
2. Map ProteinGym identifiers to UniProt accessions.
3. Load DisProt region annotations.
4. Intersect ProteinGym proteins with DisProt proteins.
5. Parse single amino-acid substitutions (for example, `A25V`).
6. Validate mutation positions against the assay target sequence.
7. Label each mutation as `IDR` or `ordered` based on DisProt residue ranges.
8. Standardize assay scores within each assay to create `DMS_score_z`.

### Feature engineering
I first used a baseline set of mutation-centric features:

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

For the bonus extension, I added local sequence-context features computed from a ±5 residue window around the mutation site, including:

- local charged fraction
- positive / negative fraction
- hydrophobic fraction
- polar fraction
- aromatic fraction
- glycine/proline fraction
- local net charge
- local mean hydropathy
- a simple low-complexity proxy

### Modeling workflow
The predictive task was:

**Predict assay-standardized mutation effect (`DMS_score_z`) from mutation/site features.**

I used grouped cross-validation by protein (`GroupKFold`) so that evaluation was protein-aware rather than allowing within-protein leakage. The baseline models compared were:

1. `mutation_only`
2. `mutation_plus_region`
3. `mutation_plus_region_plus_context` (bonus extension)

Performance was evaluated using:

- Spearman correlation
- RMSE
- R²

and was reported overall as well as separately for IDR and ordered subsets.

## Route Design and What Was Completed
### Planned route idea
The original route asked whether the “handholds” that work on ordered protein structure, especially structure-derived confidence signals, become less useful in IDRs, and whether mutation-effect distributions differ in disordered regions.

### Completed successfully
- Built a ProteinGym × DisProt benchmark
- Mapped mutation positions to IDR vs ordered regions
- Compared mutation-effect distributions in the two region types
- Trained grouped cross-validated predictive baselines
- Evaluated model performance separately for IDR and ordered mutations
- Performed balanced subsampling and protein-level robustness analyses
- Added a bonus extension using local sequence-context features

### Attempted but not completed successfully
- Structure-derived feature integration (especially pLDDT / AlphaFold-derived features)

This means the final project answers the main comparative and predictive questions, but does **not** support strong conclusions about structure-derived features in the final benchmark.

## Key Results
### 1. Mutation-effect distributions differ between IDRs and ordered regions
In the final intersected benchmark:

- IDR mean `DMS_score_z` ≈ 0.031
- ordered mean `DMS_score_z` ≈ -0.010
- IDR median `DMS_score_z` ≈ 0.274
- ordered median `DMS_score_z` ≈ 0.181

This indicates that, on average, mutations in IDRs were slightly more tolerated than mutations in ordered regions.

### 2. The pooled IDR-vs-ordered difference is robust to class imbalance
I performed balanced subsampling because the benchmark contained many more ordered than IDR variants. The result remained positive:

- balanced mean difference (`IDR - ordered`) ≈ 0.0409 ± 0.0078
- 95% interval for balanced mean difference ≈ [0.0276, 0.0567]
- balanced median difference ≈ 0.0941 ± 0.0084
- 95% interval for balanced median difference ≈ [0.0831, 0.1108]

So the shift toward more tolerated IDR mutations was not simply caused by having more ordered variants.

### 3. The protein-level effect is heterogeneous
Among proteins with both region types represented:

- 12 proteins had both IDR and ordered mutations
- 7 showed positive `IDR - ordered` median differences
- 5 showed negative differences
- Wilcoxon paired p-value ≈ 0.733

This suggests the pooled IDR effect is **not universal across proteins**. Some proteins show more tolerated IDR mutations, while others show the opposite.

### 4. Mutation-effect prediction is worse in IDRs than in ordered regions
For the baseline predictive models, Spearman correlation was consistently lower in IDRs than in ordered regions.

Approximate mean Spearman values:

- `mutation_only`: IDR ≈ 0.122, ordered ≈ 0.221
- `mutation_plus_region`: IDR ≈ 0.110, ordered ≈ 0.227

This supports the idea that mutation effects are harder to rank/predict in IDRs than in ordered protein regions.

### 5. A coarse IDR/ordered label alone adds very little
Adding only the `region_type` label produced very little overall improvement compared with mutation-only features.

This suggests that a coarse “disordered vs ordered” label is too simple to capture the sequence determinants that matter for mutation effects.

### 6. Bonus extension: local sequence context substantially improves prediction
I added a third model, `mutation_plus_region_plus_context`, using ±5 residue local sequence-context features.

This improved mean Spearman substantially:

- IDR: from ≈ 0.110 to ≈ 0.144
- all variants: from ≈ 0.208 to ≈ 0.256
- ordered: from ≈ 0.227 to ≈ 0.292

The improvement over `mutation_plus_region` was approximately:

- IDR: +0.033
- all: +0.048
- ordered: +0.065

This bonus result suggests that **local sequence environment carries useful predictive information beyond mutation identity and a coarse region label**.

## Interpretation
The final picture is more nuanced than a simple “IDRs are different” claim.

What the results support:

- IDR mutations are somewhat more tolerated on average in the pooled benchmark.
- This pooled difference is robust to class imbalance.
- Mutation-effect prediction is harder in IDRs than in ordered regions.
- A coarse IDR/ordered label is not enough to improve prediction much.
- Local sequence context provides substantial additional signal.

What the results do **not** support strongly:

- that the IDR-vs-ordered effect is universal across proteins
- that structure-derived features were successfully tested in the final benchmark
- that sequence context helps IDRs more than ordered regions specifically

In fact, the context bonus improved prediction in **both** region types, with a larger gain in ordered regions in this benchmark.

## Limitations
- The benchmark overlap between ProteinGym and DisProt was modest.
- Only 12 proteins had both IDR and ordered mutation subsets available for paired protein-level analysis.
- Structure-derived features were attempted but did not populate successfully.
- The final analysis focuses on single substitutions and does not incorporate indels or more complex mutational combinations.
- The project uses a pooled benchmark and relatively simple models rather than protein-specific or deep sequence models.

## Next Steps
If I extended the project further, I would prioritize:

1. Fixing structure-file mapping so pLDDT and related confidence features can be tested directly.
2. Expanding the sequence-context model with richer composition or motif-aware features.
3. Running protein-level or assay-level models to better separate global trends from protein-specific effects.
4. Testing whether the same conclusions hold under stricter external validation or additional DMS benchmarks.

## Reproducibility / How to Run
### Main notebook
Run the final notebook in Google Colab or Jupyter.

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

## Repository Contents
Recommended repository structure:

```text
.
├── README.md
├── notebooks/
│   └── disordered_wall_pipeline_final.ipynb
├── figures/
│   ├── mutation_effect_distributions.png
│   ├── model_performance_by_region.png
│   ├── balanced_subsampling.png
│   └── bonus_context_model.png
├── results/
│   ├── model_summary.csv
│   ├── protein_level_summary.csv
│   └── bonus_context_summary.csv
└── data/
    └── README_data_notes.md
```

## Key Figures / Artifacts to Link
Link these clearly from the repo:

- final notebook
- key figures used in the writeup
- model summary tables
- any exported CSV results

## Route Prompt
**The Disordered Wall**

*A route that ventures off the well-mapped face of ordered protein domains and into the loose, unpredictable terrain of intrinsically disordered regions. The climber asks whether the handholds that work on solid rock become useless on the crumbling IDR face — and whether mutation-effect scores follow different distributions in disorder.*
