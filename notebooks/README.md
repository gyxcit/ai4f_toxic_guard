# Notebooks

This folder contains the four Jupyter notebooks that reproduce the full
pipeline from raw data to publication figures.

---

## Files

| Notebook | Role | Inputs | Outputs |
|---|---|---|---|
| `ai4fi-data.ipynb` | Data pipeline | Raw Uniswap (Dune) and Binance klines | `aligned_data.parquet` with all 4 toxicity scores precomputed |
| `ai4f-space.ipynb` | Environment + training (preliminary) | `aligned_data.parquet` | Initial PPO training experiments on a subset of windows |
| `aif4-space2.ipynb` | Full ablation training | `aligned_data.parquet`, `rolling_windows.parquet` | 105 PPO runs (7 configs × 5 windows × 3 seeds), per-run results CSV, 35 trained checkpoints |
| `aif4-space-figure.ipynb` | Statistical analysis and figures | Per-run CSV from `aif4-space2.ipynb` | The four publication figures (PDF + PNG, 300 DPI), Pearson correlation matrix between toxicity scores |

---

## Recommended execution order

```
1. ai4fi-data.ipynb           ← run once to build aligned_data.parquet
2. aif4-space2.ipynb          ← the main 105-run ablation (~6.5h on 2× T4)
3. aif4-space-figure.ipynb    ← post-hoc analysis and figures (~2 min)
```

The notebook `ai4f-space.ipynb` is preserved for transparency: it documents
the preliminary single-window experiments used to validate the environment
design and PPO hyperparameters before the full ablation. It is not required
to reproduce the paper's results.

---

## Quick reproduction

```bash
pip install -r ../requirements.txt
jupyter notebook ai4fi-data.ipynb
# then aif4-space2.ipynb
# then aif4-space-figure.ipynb
```

---

## `ai4fi-data.ipynb` — what it does

1. Loads raw Uniswap v3 hourly aggregates from Dune Analytics
2. Loads Binance ETHUSDT 1h spot klines
3. Aligns both series on UTC hour boundaries
4. Computes 24h and 7d rolling volatilities from Binance log-returns
5. Computes the four toxicity scores defined in Section 4 of the paper
6. Drops rows with NaN in critical columns
7. Writes `aligned_data.parquet` (24,019 hourly rows)

---

## `aif4-space2.ipynb` — what it does

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
```

Two T4 GPUs are dispatched in parallel via
`concurrent.futures.ThreadPoolExecutor`: configurations R1–R4 run on
`cuda:0` and R5–R7 on `cuda:1`. Intermediate progress is written to
`results_checkpoint.csv` after each completed run, allowing safe resumption
from interruption.

The notebook also logs, for the held-out test episodes:

- Episode rewards per training step (used for learning curves)
- Action–toxicity correlations on R5_volsize (the r = +0.41 figure cited
  in the paper)

---

## `aif4-space-figure.ipynb` — what it does

1. Loads the per-run results from `aif4-space2.ipynb`
2. Aggregates per configuration: mean, std, win count, win rate
3. Runs Fisher exact tests on win rates (vs R1_baseline)
4. Runs Welch t-tests on mean excess return (vs R1_baseline)
5. Computes CVaR$_{10\%}$ and CVaR$_{25\%}$ per configuration
6. Decomposes performance by dominant market regime (calm / normal /
   stressed)
7. Computes the pairwise Pearson correlation matrix between the four
   toxicity scores on the full 24,019-hour dataset
8. Generates the four publication figures (PDF + PNG, 300 DPI)

---

## `ai4f-space.ipynb` — what it does (preliminary)

A scaled-down version of `aif4-space2.ipynb` covering a single rolling
window (W1) and a single seed, used to validate the environment design
(`UniswapV3LPEnvBaseline` and `UniswapV3LPEnvToxicityAware`) and to
calibrate the PPO hyperparameters before the full 105-run ablation. Not
required to reproduce the paper's reported numbers.

---

## Compute requirements

- Python 3.10+ with the packages in `../requirements.txt`
- For `aif4-space2.ipynb`: 2× NVIDIA T4 GPU recommended
  (Kaggle free tier provides this out of the box). Single GPU works but
  doubles the wall-clock time.
- For `ai4fi-data.ipynb` and `aif4-space-figure.ipynb`: any CPU is sufficient

A single PPO training run takes approximately 7 minutes on a T4 GPU; the
full 105-run ablation completes in approximately 6.5 hours on two T4 GPUs.

---

## Tips for reproducing on Kaggle

1. Create a new Kaggle notebook
2. Attach the four datasets from `../data/` as input
3. Set Accelerator → "GPU T4 ×2"
4. Set Persistence → "Variables and Files"
5. Run all cells; intermediate checkpoints are saved automatically

If the Kaggle session is interrupted, the resume cell at the top of
`aif4-space2.ipynb` will detect existing entries in
`results_checkpoint.csv` and skip already-completed runs.
