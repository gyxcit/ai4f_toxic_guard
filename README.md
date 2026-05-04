# Toxicity-Aware Reinforcement Learning for Liquidity Provisioning on Uniswap v3

> Anonymous code release for the NeurIPS 2026 submission
> *"Toxicity-Aware Reinforcement Learning for Liquidity Provisioning on Uniswap v3:
> A Systematic Ablation"*.

This repository contains the full pipeline, dataset, and analysis notebook for a
factorial ablation of seven observation configurations across five rolling
windows and three random seeds (105 PPO training runs in total) on the
WETH/USDC 0.05% Uniswap v3 pool, May 2021 to January 2024.

---

## Quick reproduction

```bash
git clone <this-repo>
cd toxicity-aware-rl-uniswap
pip install -r requirements.txt
jupyter notebook notebooks/02_train_and_evaluate.ipynb
```

The full pipeline runs in approximately 6.5 hours on two NVIDIA T4 GPUs
(Kaggle free tier).

---

## Repository structure

| Folder | Content |
|---|---|
| `notebooks/` | Jupyter notebooks for the full pipeline |
| `data/` | Aligned hourly dataset and raw Uniswap/Binance sources |
| `figures/` | The 4 publication figures (PDF + PNG) |

---

## Notebooks

| Notebook | Purpose |
|---|---|
| `01_data_pipeline.ipynb` | Reconstruction of the aligned hourly dataset from raw Dune Analytics and Binance sources |
| `02_train_and_evaluate.ipynb` | Trains 105 PPO agents (7 configs × 5 windows × 3 seeds) and evaluates them on held-out test windows |
| `03_analysis_and_figures.ipynb` | Statistical analysis (Fisher exact, Welch t-test, CVaR), regime decomposition, and generation of the 4 paper figures |

Each notebook is self-contained and can be re-run independently if intermediate
artifacts are present.

---

## Datasets

| File | Description |
|---|---|
| `data/aligned_data.parquet` | 24,019 hourly observations with all 4 toxicity scores precomputed |
| `data/uniswap_dune.parquet` | Raw Uniswap v3 WETH/USDC 5bps hourly aggregates from Dune Analytics |
| `data/binance_eth_usdt_1h.parquet` | Binance ETHUSDT 1h spot klines (May 2021 – Jan 2024) |
| `data/rolling_windows.parquet` | Definitions of the 11 rolling train/test split windows |
| `data/dune_query.sql` | The exact SQL query used to extract Uniswap pool data from Dune |

The aligned dataset spans **2021-05-05 22:00 UTC** to **2024-01-31 23:00 UTC** and
covers $428.57B of cumulative pool volume across 5,964,951 swaps.

---

## Tested configurations and main results

| ID | Toxicity features | Win rate (15 runs) | CVaR$_{10\%}$ |
|---|---|---|---|
| R1_baseline | none (Xu & Brini reproduction) | 87% | -$11.84 |
| R2_lvr | tox_lvr | 93% | -$14.12 |
| R3_spread | tox_spread | 73% | -$26.33 |
| R4_realized | tox_realized | 80% | -$10.61 |
| **R5_volsize** | **tox_volsize** | **100%** | **+$4.00** |
| R6_lvr+spread | tox_lvr + tox_spread | 53% | -$27.31 |
| R7_all | all four toxicity features | 67% | -$63.09 |

**R5_volsize** is the only configuration that achieves a strictly positive
CVaR$_{10\%}$ and a perfect win rate across all five test windows. See the
paper for the full analysis.

---

## Note on per-run logs and trained checkpoints

The per-run results CSVs (`results_final.csv`, `cvar_results.csv`,
`learning_curves.csv`, `actions_log_R5.csv`) and the 35 trained PPO model
checkpoints (one per configuration × window for seed=42) are **not included in
this anonymous release** due to anonymous-repository size constraints.
They will be made available upon acceptance, when the repository is moved to a
public GitHub URL with full attribution.

The numerical values reported in the paper (Tables 4–7 and Figures 1–4) can be
fully reproduced by re-running `notebooks/02_train_and_evaluate.ipynb` followed
by `notebooks/03_analysis_and_figures.ipynb` on the included datasets.

---

## Toxicity scores

The four scores are defined in `notebooks/01_data_pipeline.ipynb`:

```
tox_spread   = |P_uni - P_bin| / P_bin × 10^4
tox_lvr      = sqrt(0.5) × σ_24h × |ΔP/P| × 10^4
tox_realized = (V_t / median(V)) × tox_spread
tox_volsize  = log10(1 + p95_swap_size) × tox_spread
```

All scores are validated to scale monotonically across calm/normal/stressed
market regimes by factors of 2.79× to 8.46×.

---

## Compute requirements

- **Recommended:** 2× NVIDIA T4 GPUs (Kaggle free tier provides this)
- **Wall-clock:** ~6.5 hours for the full 105-run ablation
- **Memory:** 16 GB RAM sufficient
- **Disk:** ~500 MB for data + intermediate outputs

A single PPO training run takes approximately 7 minutes on a T4 GPU.

---

## Citation

This repository accompanies a paper currently under double-blind review at
NeurIPS 2026. The full citation will be provided upon acceptance.

---

## License

- **Code** (notebooks): MIT License
- **Data** (parquet files): CC-BY 4.0

See `LICENSE` for full terms.

---

## Dependencies

See `requirements.txt`. Key packages:

- `stable-baselines3==2.3.2`
- `gymnasium==0.29.1`
- `pandas`, `numpy`, `scipy`, `matplotlib`, `seaborn`
- `torch>=2.2`