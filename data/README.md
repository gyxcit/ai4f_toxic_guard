# Datasets

This folder contains the four parquet files and the SQL query needed to
reproduce the full data pipeline.

---

## Files

| File | Rows | Description |
|---|---|---|
| `aligned_data.parquet` | 24,019 | Hourly observations with all 4 toxicity scores precomputed; this is the file consumed by `02_train_and_evaluate.ipynb` |
| `uniswap_dune.parquet` | ~24k | Raw Uniswap v3 WETH/USDC 5bps hourly aggregates from Dune Analytics: volume, VWAP, swap-size percentiles |
| `binance_eth_usdt_1h.parquet` | ~24k | Binance ETHUSDT 1-hour spot klines (close prices, volatility) |
| `rolling_windows.parquet` | 11 | Definitions of the 11 rolling train/test windows (train_start, train_end, test_start, test_end) |
| `dune_query.sql` | 1 | The exact SQL query used to extract Uniswap pool data from Dune Analytics |

---

## Time coverage

- **Start:** 2021-05-05 22:00 UTC (pool genesis)
- **End:** 2024-01-31 23:00 UTC
- **Frequency:** 1 hour
- **Total rows:** 24,019 aligned observations

Cumulative pool activity over the period:

- $428.57B in cumulative volume
- 5,964,951 swaps
- $214.29M in cumulative LP fees

---

## `aligned_data.parquet` schema

| Column | Type | Description |
|---|---|---|
| `timestamp` | datetime[ns, UTC] | Hour start (index) |
| `binance_price` | float64 | Binance ETHUSDT close at hour end |
| `uniswap_price` | float64 | Volume-weighted average pool price during the hour |
| `price_deviation_bps` | float64 | (P_uni − P_bin) / P_bin × 10^4 |
| `vol_24h` | float64 | 24-hour rolling volatility of Binance log-returns |
| `vol_7d` | float64 | 7-day rolling volatility (used for regime classification) |
| `uniswap_volume_usd` | float64 | Pool volume in USD during the hour |
| `uniswap_n_swaps` | int64 | Number of swaps in the hour |
| `uniswap_p95_swap_size_usd` | float64 | 95th-percentile swap size in USD |
| `tox_spread` | float64 | Spread-based toxicity (basis points) |
| `tox_lvr` | float64 | Analytical LVR proxy (Milionis et al. 2022) |
| `tox_realized` | float64 | Volume-weighted toxicity |
| `tox_volsize` | float64 | Swap-size signature toxicity |

---

## Reconstruction from raw sources

If you wish to rebuild `aligned_data.parquet` from scratch:

1. Run `dune_query.sql` on Dune Analytics → save as `uniswap_dune.parquet`
2. Pull Binance ETHUSDT 1h klines → save as `binance_eth_usdt_1h.parquet`
3. Run `notebooks/01_data_pipeline.ipynb` → produces `aligned_data.parquet`

The pipeline notebook handles timezone alignment, NaN imputation for
swap-size fields when missing, and the four toxicity-score computations.

---

## License

These datasets are released under the Creative Commons Attribution 4.0
International License (CC-BY 4.0). Underlying data sources are public:

- **Uniswap v3** on-chain data: queried via Dune Analytics
  (https://dune.com)
- **Binance ETHUSDT** klines: public market-data API
  (https://www.binance.com/en/binance-api)