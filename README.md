# CDRPipe Comparative Analysis

This repository contains the manuscript-specific comparative analyses and figure-generation code for the CDRPipe platform evaluation. It was split out of the main CDRPipe repository so the package/application code and the manuscript reproduction workflow can evolve independently.

Original package repository: https://github.com/enockniyonkuru/drug_repurposing

Manuscript analysis repository: https://github.com/enockniyonkuru/cdrpipe-comparative-analysis

---

## What Lives Where

Use this repository for manuscript case studies, batch analyses, curated analysis tables, and figure generation. Use `drug_repurposing` for the core R package, Shiny app, general pipeline scripts, and package documentation.

| Repository | Purpose |
|------------|---------|
| `drug_repurposing` | Source for the `CDRPipe` R package and general-purpose pipeline |
| `cdrpipe-comparative-analysis` | Reproducible manuscript analyses comparing CMap and TAHOE |

This repository imports CDRPipe as a normal R package:

```r
library(CDRPipe)
```

Install `CDRPipe` from the original repository before running any batch analyses.

---

## Repository Structure

```
cdrpipe-comparative-analysis/
├── shared/                            # Shared inputs, provenance, and batch runners
│   ├── gene_id_conversion_table.tsv
│   ├── shared_drugs_cmap_tahoe.csv
│   ├── figure_provenance_manifest.csv
│   └── scripts/execute/
├── drug_signatures/                   # CMap and TAHOE preparation/visualization
│   ├── data/
│   ├── scripts/
│   └── figures/
├── drug_evidence/                     # Known drug-disease evidence processing
├── creeds/                            # CREEDS 233-disease manuscript analysis
├── autoimmune/                        # Autoimmune known-drug recovery case study
└── endometriosis/                     # Endometriosis 19-signature case study
```

Study-specific documentation:

| Study | README | Main configs |
|-------|--------|--------------|
| CREEDS | [creeds/README.md](creeds/README.md) | `creeds/scripts/execute/creeds_manual_config_all_avg.yml` |
| Autoimmune | [autoimmune/README.md](autoimmune/README.md) | `autoimmune/scripts/execute/autoimmune_batch_config.yml` |
| Endometriosis | [endometriosis/README.md](endometriosis/README.md) | `endometriosis/scripts/execute/*.yml` |
| Drug signatures | [drug_signatures/README.md](drug_signatures/README.md) | `drug_signatures/scripts/` |
| Drug evidence | [drug_evidence/README.md](drug_evidence/README.md) | `drug_evidence/scripts/` |
| Shared runners | [shared/scripts/README.md](shared/scripts/README.md) | `shared/scripts/execute/` |

---

## Fresh Setup From Scratch

The cleanest setup is to clone the manuscript repository and the original package repository side by side.

```bash
mkdir cdrpipe-reproduction
cd cdrpipe-reproduction

git clone https://github.com/enockniyonkuru/drug_repurposing.git
git clone https://github.com/enockniyonkuru/cdrpipe-comparative-analysis.git

cd cdrpipe-comparative-analysis
```

### 1. Install R Dependencies

Install `CDRPipe` from the original repository:

```bash
Rscript -e 'install.packages("devtools", repos = "https://cloud.r-project.org")'
Rscript -e 'devtools::install("../drug_repurposing/CDRPipe")'
```

Install additional R packages used by the manuscript scripts:

```bash
Rscript -e 'pkgs <- c("argparse", "yaml", "dplyr", "tidyr", "readxl", "ggplot2", "gridExtra", "cowplot", "patchwork", "arrow", "VennDiagram", "DiagrammeR", "DiagrammeRsvg", "rsvg", "gplots", "jsonlite", "UpSetR", "tidyverse"); missing <- setdiff(pkgs, rownames(installed.packages())); if (length(missing)) install.packages(missing, repos = "https://cloud.r-project.org")'
```

Verify that CDRPipe imports successfully:

```bash
Rscript -e 'library(CDRPipe); cat("CDRPipe loaded from:", find.package("CDRPipe"), "\n")'
```

You should see a package path and no error.

### 2. Install Python Dependencies

Use a local virtual environment for Python analysis and figure scripts:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install pandas numpy scipy pyarrow h5py matplotlib seaborn adjustText openpyxl
```

### 3. Add Large Data Files

Large drug-signature matrices are not stored in GitHub. Download the required files from the same data release used by the main repository:

Box data folder: https://ucsf.box.com/s/m54ipylmdytjsqmlp7axnabvjh2q8lwl

For CMap analyses, place:

```text
drug_signatures/data/cmap/cmap_signatures.RData
drug_signatures/data/cmap/cmap_drug_experiments_new.csv
drug_signatures/data/cmap/cmap_valid_instances_OG_015.csv
```

For TAHOE analyses, place or generate:

```text
drug_signatures/data/tahoe/tahoe_signatures.RData
drug_signatures/data/tahoe/tahoe_drug_experiments_new.csv
drug_signatures/data/tahoe/tahoe_valid_instances_OG_035.csv
```

If the Tahoe release provides `tahoe_signatures.parquet` instead of `tahoe_signatures.RData`, download `convert_parquet_to_rdata.R` from Box and run:

```bash
cd drug_signatures/data/tahoe
Rscript convert_parquet_to_rdata.R tahoe_signatures.parquet -o tahoe_signatures.RData --object-name tahoe_signatures --force
cd ../../..
```

Some figure scripts also expect Open Targets evidence at:

```text
drug_evidence/data/open_targets/known_drug_info_data.parquet
```

### 4. Create Legacy Data Path Links

Several manuscript scripts were written before this repository was split out and expect shared data under a local `data/` entry point. Create symlinks after downloading the large files:

```bash
mkdir -p data
ln -sfn ../drug_signatures/data data/drug_signatures
ln -sfn ../drug_evidence/data data/drug_evidence
ln -sfn ../shared/gene_id_conversion_table.tsv data/gene_id_conversion_table.tsv

for study in creeds autoimmune endometriosis; do
  mkdir -p "$study/data"
  ln -sfn ../../drug_signatures/data "$study/data/drug_signatures"
  ln -sfn ../../shared/gene_id_conversion_table.tsv "$study/data/gene_id_conversion_table.tsv"
done
```

These links are local setup helpers only. Do not commit them.

### 5. Verify Data Files

From the repository root:

```bash
Rscript -e 'e <- new.env(); load("drug_signatures/data/cmap/cmap_signatures.RData", envir=e); obj <- get(ls(e)[1], envir=e); cat("CMap:", class(obj), paste(dim(obj), collapse=" x "), "\n")'

Rscript -e 'e <- new.env(); load("drug_signatures/data/tahoe/tahoe_signatures.RData", envir=e); obj <- get(ls(e)[1], envir=e); cat("TAHOE:", class(obj), paste(dim(obj), collapse=" x "), "\n"); print(head(obj[, 1:10]))'
```

The Tahoe preview should show numeric entries, not all `NA`.

---

## Reproducing the Manuscript Workflow

There are two levels of reproduction:

| Level | Goal | Inputs required |
|-------|------|-----------------|
| Figure-only regeneration | Recreate manuscript figures from committed or curated analysis tables | Figure input tables plus plotting dependencies |
| Full end-to-end rerun | Recompute CDRPipe results, analysis tables, and figures | Large CMap/TAHOE matrices, Open Targets evidence, CDRPipe package, and longer runtime |

The figure provenance file is:

```text
shared/figure_provenance_manifest.csv
```

Use it to trace each figure family to its generator script and required direct inputs.

---

## Full End-to-End Rerun

Run these commands from the `cdrpipe-comparative-analysis` repository root after completing setup.

### 1. CREEDS 233-Disease Batch

```bash
Rscript shared/scripts/execute/run_batch_from_config.R \
  --config_file creeds/scripts/execute/creeds_manual_config_all_avg.yml
```

This writes CDRPipe outputs to:

```text
creeds/results/manual_standardized_all_diseases_results/
creeds/analysis/manual_standardized_all_diseases_analysis/
```

### 2. Autoimmune Batch

```bash
Rscript shared/scripts/execute/run_batch_from_config.R \
  --config_file autoimmune/scripts/execute/autoimmune_batch_config.yml
```

This writes outputs to the autoimmune results and analysis directories defined in the config.

### 3. Endometriosis Batches

```bash
Rscript shared/scripts/execute/run_batch_from_config.R \
  --config_file endometriosis/scripts/execute/single_cell_batch_config.yml

Rscript shared/scripts/execute/run_batch_from_config.R \
  --config_file endometriosis/scripts/execute/microarray_strict_config.yml

Rscript shared/scripts/execute/run_batch_from_config.R \
  --config_file endometriosis/scripts/execute/all_19_signatures_config.yml
```

These runs can be long, especially for TAHOE. For smoke testing, edit the relevant config to process a small disease range with `start_from_disease` and `end_at_disease`.

---

## Regenerating Figures

Run these after the required input tables exist. Some inputs are committed in this repository; others are produced by the batch and analysis steps above. If a script reports a missing file, check `shared/figure_provenance_manifest.csv` for the expected input.

### Platform Comparison Figures

```bash
Rscript drug_signatures/scripts/visualization/generate_platform_comparison.R
```

Primary outputs:

```text
figures/platform_comparison/
```

Tracked manuscript copies are stored under:

```text
drug_signatures/figures/platform_comparison/
```

### CREEDS Figures

```bash
Rscript creeds/scripts/visualization/plot_disease_signature_filtering.R
Rscript creeds/scripts/visualization/plot_analysis_across_diseases.R
python creeds/scripts/visualization/plot_drug_class_distributions.py
python creeds/scripts/visualization/plot_biological_concordance.py
python creeds/scripts/visualization/plot_concordance_metrics.py
```

Primary outputs:

```text
creeds/figures/
```

### Autoimmune Figures

```bash
python autoimmune/scripts/visualization/generate_case_study_autoimmune.py
python autoimmune/scripts/visualization/create_drug_consistency_figures.py
python autoimmune/scripts/visualization/create_separate_panels.py
```

Primary outputs:

```text
autoimmune/figures/
```

### Endometriosis Figures

```bash
Rscript endometriosis/scripts/visualization/generate_case_study_endometriosis.R
python endometriosis/scripts/visualization/plot_disease_signature_info.py
Rscript endometriosis/scripts/analysis/generate_upset_plots.R
Rscript endometriosis/scripts/analysis/plot_drug_overlaps_upset_v2.R
```

Primary outputs:

```text
endometriosis/figures/
endometriosis/analysis/
```

---

## Important Reproducibility Notes

- Run commands from the repository root unless a study-specific README says otherwise.
- Keep `drug_repurposing` and `cdrpipe-comparative-analysis` as sibling directories if using the local CDRPipe install command above.
- Large `.RData`, `.rds`, and `.parquet` files are intentionally not tracked in GitHub.
- The batch configs use local relative paths; the symlink setup above keeps legacy paths working after the repository split.
- TAHOE runs are much heavier than CMap runs and may require substantial memory and runtime.
- Existing committed figures are manuscript outputs; rerunning scripts may overwrite them locally.

---

## Data Availability

| Data | Expected location | Notes |
|------|-------------------|-------|
| CMap signature matrix | `drug_signatures/data/cmap/cmap_signatures.RData` | Download from Box data release |
| CMap metadata | `drug_signatures/data/cmap/cmap_drug_experiments_new.csv` | Included or downloadable from Box |
| TAHOE signature matrix | `drug_signatures/data/tahoe/tahoe_signatures.RData` | Generate from Tahoe parquet if needed |
| TAHOE metadata | `drug_signatures/data/tahoe/tahoe_drug_experiments_new.csv` | Included or downloadable from Box |
| Open Targets evidence | `drug_evidence/data/open_targets/known_drug_info_data.parquet` | Download from Open Targets or project data release |
| Gene mapping | `shared/gene_id_conversion_table.tsv` | Included in this repository |
| Disease signatures | `creeds/data/`, `autoimmune/data/`, `endometriosis/data/` | Included where small enough for GitHub |

Primary external sources:

| Source | Link |
|--------|------|
| Main CDRPipe repository | https://github.com/enockniyonkuru/drug_repurposing |
| Box data release | https://ucsf.box.com/s/m54ipylmdytjsqmlp7axnabvjh2q8lwl |
| CMap | https://www.broadinstitute.org/connectivity-map-cmap |
| TAHOE | https://huggingface.co/datasets/tahoebio/Tahoe-100M |
| Open Targets | https://platform.opentargets.org/downloads |
