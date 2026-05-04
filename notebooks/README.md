# Notebooks

This folder contains the three Jupyter notebooks that reproduce the full
pipeline from raw data to publication figures.

---

## Files

| Notebook | Inputs | Outputs | Wall-clock |
|---|---|---|---|
| `01_data_pipeline.ipynb` | `data/uniswap_dune.parquet`, `data/binance_eth_usdt_1h.parquet` | `data/aligned_data.parquet` | ~2 min |
| `02_train_and_evaluate.ipynb` | `data/aligned_data.parquet`, `data/rolling_windows.parquet` | `results/results_final.csv`, `results/learning_curves.csv`, `results/actions_log_R5.csv`, 35 PPO checkpoints in `models/` | ~6.5 hours on 2× T4 GPU |
| `03_analysis_and_figures.ipynb` | `results/results_final.csv` | `results/cvar_results.csv`, `figures/fig1_cvar.pdf` through `fig4_corr.pdf` | ~2 min |

---

## How to run

### Option 1 — Full pipeline from scratch

```bash
pip install -r requirements.txt
jupyter notebook notebooks/01_data_pipeline.ipynb
# then 02_train_and_evaluate.ipynb
# then 03_analysis_and_figures.ipynb
```

### Option 2 — Skip data pipeline (recommended)

`aligned_data.parquet` is already provided in `data/`. You can therefore
start directly with `02_train_and_evaluate.ipynb`.

### Option 3 — Skip training (figures only)

This requires `results/results_final.csv` (released upon acceptance). With
that file in place, `03_analysis_and_figures.ipynb` regenerates all figures
in approximately 2 minutes.

---

## `01_data_pipeline.ipynb` — what it does

1. Loads raw Uniswap v3 hourly aggregates (Dune Analytics extract)
2. Loads Binance ETHUSDT 1h klines
3. Aligns both series on UTC hour
4. Computes 24h and 7d rolling volatilities from Binance log-returns
5. Computes the four toxicity scores defined in Section 4 of the paper
6. Drops rows with NaN in critical columns
7. Writes `aligned_data.parquet`

---

## `02_train_and_evaluate.ipynb` — what it does

The main notebook. Trains 105 PPO agents using the following grid:

- **Configurations** (7): R1_baseline, R2_lvr, R3_spread, R4_realized,
  R5_volsize, R6_lvr+spread, R7_all
- **Rolling windows** (5): W1, W3, W6, W9, W11 (uniformly spaced over the
  11 feasible windows)
- **Random seeds** (3): 42, 123, 2024

Each PPO run uses the hyperparameters of Table 3 in the paper:

```
learning_rate    = 3e-4
n_steps          = 512
batch_size       = 64
n_epochs         = 10
gamma            = 0.99
gae_lambda       = 0.95
clip_range       = 0.2
ent_coef         = 0.01
policy_kwargs    = MLP [128, 128]
total_timesteps  = 80,000
device           = cuda:0 (R1–R4) / cuda:1 (R5–R7)
```

Two T4 GPUs are dispatched in parallel via `concurrent.futures.ThreadPoolExecutor`.
Intermediate progress is written to `results/results_checkpoint.csv` after
each completed run, allowing safe resumption from interruption.

The notebook also logs:

- Episode rewards per training step (for learning curves)
- Action-toxicity correlations on the held-out test window for R5_volsize
  (the r = +0.41 figure cited in the paper)

---

## `03_analysis_and_figures.ipynb` — what it does

1. Loads `results_final.csv` (105 rows: window × config × seed)
2. Aggregates per configuration: mean, std, win count, win rate
3. Runs Fisher exact tests on win rates (vs R1_baseline)
4. Runs Welch t-tests on mean excess return (vs R1_baseline)
5. Computes CVaR$_{10\%}$ and CVaR$_{25\%}$ per configuration
6. Decomposes performance by dominant market regime
7. Generates the four publication figures (PDF + PNG, 300 DPI)
8. Outputs the LaTeX table for Section 6 of the paper

---

## Compute requirements

- Python 3.10+ with the packages in `../requirements.txt`
- For `02_train_and_evaluate.ipynb`: 2× NVIDIA T4 GPU recommended
  (Kaggle free tier provides this out of the box). Single GPU works but
  doubles the wall-clock time.
- For `01` and `03`: any CPU is sufficient

---

## Tips for reproducing on Kaggle

1. Create a new Kaggle notebook
2. Attach the four datasets from `data/` as input
3. Set Accelerator → "GPU T4 ×2"
4. Set Persistence → "Variables and Files"
5. Run all cells; intermediate checkpoints are saved automatically

If the session is interrupted, the resume cell at the top of
`02_train_and_evaluate.ipynb` will detect existing entries in
`results_checkpoint.csv` and skip already-completed runs.