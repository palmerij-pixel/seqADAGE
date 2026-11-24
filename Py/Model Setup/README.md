# Does ensembling change what SHAP explains?

Code for my CSC 488 thesis at Union College (advisor: Prof. Georgia Doing), *Ensembles of Neural Networks
Give More Reproducible Explanations of Gene Expression*.

The pipeline trains 125 ADAGE models (denoising autoencoders) on an *E. coli* gene expression compendium, in five
independent groups of 25. It then builds one ensemble model (eADAGE) per group, explains all 130 models with SHAP,
and tests whether the explanations of independently built eADAGEs agree more than those of single models. It uses the
lab's seqADAGE code from this repository (`Py/adage` and `ensemble_construction`).

## Requirements

**Python 3.9.6** with these packages (the versions used for the thesis):

```bash
pip install tensorflow==2.19.0 keras==3.9.0 keras-tuner==1.4.7 shap==0.47.1 \
            numpy==2.0.2 scipy==1.13.1 pandas==2.3.3 statsmodels==0.14.6 \
            matplotlib==3.9.4 plotnine==0.13.6
```

**R 4.5.3** with the packages ClusterR (1.3.7), ff (4.5.3) and readr (2.2.0), for the consensus clustering.

All computation ran on the ACES cluster at Texas A&M University (through ACCESS), using Slurm to run one job per
model. No GPU is needed.

## Folder layout

The notebooks expect one project folder that contains the data, a copy of this repository (`Jordin` branch) and the
results:

```
CSC-488-Code/
├── data/
│   └── Copy of ecoli_MG1655plus_genome_part2_log_counts_norm_01.csv
├── seqADAGE/          # this repository; 00_setup.ipynb clones it here
└── runs/              # created by the notebooks; every result is saved here
```

The compendium (6,390 genes x 16,122 samples, log-transformed and scaled 0 to 1) was provided by the Doing lab and is
not included in this repository.

Each notebook starts with a settings cell. The only line to change for another setup is
`PROJECT = os.path.expandvars("$SCRATCH/CSC-488-Code")`.

## Running the pipeline

Run the notebooks in order. Each one reads the previous notebook's results from `runs/`. Results that are already
saved are skipped, so a notebook can be rerun safely. Set `FORCE = True` in its settings cell to redo the work.

| Notebook | What it does | How to run it | Time |
|---|---|---|---|
| `00_setup` | Clones the lab's code, prints package versions, checks the data | interactively | minutes |
| `01_tune` | Hyperband search for the training hyperparameters (90 settings) | interactively | about 2 h |
| `02_train_models` | Trains the 125 models (model N uses seed 171 + N), then plots the training checks | `train_models.slurm` (array 0-124), then rerun interactively for the plots | 35-125 min per model |
| `03_build_eadage` | Builds one eADAGE per group: weighted Pearson correlation, consensus clustering into 450 clusters, mean of each cluster | `build_eadage.slurm` (array 0-4); needs about 32 GB of memory | about 30 min per group |
| `04_shap` | Explains all 130 models with SHAP DeepExplainer (100 background and 50 explained samples, the same for every model) | `shap_all.slurm` (array 0-129) | about 4 min per model |
| `05_compare` | Matches nodes between models and compares their SHAP gene rankings (Spearman correlation, top-50 overlap) | interactively | 10-20 min |
| `06_tests_and_figures` | Paired t-tests over the independent groups (thesis Table 3) and the result figures | interactively | minutes |

The Slurm scripts run a notebook as a batch job; array task N handles model N (or group N, or SHAP task N).

## Where the thesis results come from

| Thesis | Notebook and file |
|---|---|
| Table 1 (hyperparameters) | `01_tune`: `runs/tuning/best_hparams.json` |
| Section 4.1, training checks | `02_train_models`: `runs/training_check/` |
| Section 4.2, quiet nodes and cluster sizes | `02_train_models`, `03_build_eadage`, `04_shap` |
| Sections 4.3 to 4.5, median agreements | `05_compare`: `runs/results/summary.csv` and `summary_without_quiet.csv` |
| Table 3 and the result figures | `06_tests_and_figures`: `runs/figures/` |

## Changes to the lab's code

The `Jordin` branch includes three corrections to the lab's code, which all results use:

1. The encoder now applies its activation function (`tanh`) (`LinkedAE`, `SeqAdage`, `AdageHyperModel`).
2. Training rebuilds the clean data from the noisy copy, not the other way round (`SeqAdage.train_model_variable`).
3. Consensus clustering keeps its random state, so that the 10 subsamples differ, and saves each node's cluster
   (`ensemble_construction/ensemble_construction_p_dcc.R`).
