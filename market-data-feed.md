---
name: market-data-feed
description: Fetches raw OHLCV data and real-time bid/ask ticks for Gold (XAUUSD!) and its cross (EURUSD!) using MT5, specifically tailored for the GOLD SNIPER v9.0 RL agent. Provides the exact dataframes required by the GQRAgent feature extractor.
version: 1.0.0
---

# Purpose

Supplies the **GOLD SNIPER v9.0** engine with live market data. It retrieves multi-timeframe candles (M1, M5, M15) for the primary symbol and its cross-asset reference, along with current spreads and pip sizes. This skill acts as the direct data bridge between MetaTrader 5 and the `GQRAgent`'s feature engineering layer.

---

# Inputs

The skill expects the following parameters when invoked by the `GQRAgent` (or the main loop):

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `symbol` | string | Yes | `XAUUSD!` | Primary trading instrument (Gold vs USD). |
| `cross_symbol` | string | No | `EURUSD!` | Cross-instrument for fusion (replaces DXY). |
| `timeframe_m1_bars` | integer | No | `200` | Number of 1-minute candles to fetch. |
| `timeframe_m5_bars` | integer | No | `120` | Number of 5-minute candles to fetch. |
| `timeframe_m15_bars` | integer | No | `80` | Number of 15-minute candles to fetch. |
| `include_bid_ask` | boolean | No | `true` | If true, returns current `bid`/`ask` and `spread_pips`. |

---

# Execution

1.  **MT5 Connection Check**: Ensures the MetaTrader 5 terminal is initialized and the specified symbols are available.
2.  **Data Fetching (Async)**:
    - Calls `fetch_ohlcv_sync` (or its async wrapper) using `mt5.copy_rates_from_pos`.
    - Fetches **M1**, **M5**, and **M15** data for the `symbol` (Gold).
    - Fetches **M1**, **M5**, and **M15** data for the `cross_symbol` (EURUSD).
3.  **Pip Calculation**:
    - Uses the `PipInfo.pip_size(symbol)` method exactly as defined in Section 0 of the `v9.0` code to determine the correct decimal pip value (handles 2/3/5 digit brokers automatically).
4.  **Tick Snapshot**:
    - Calls `mt5.symbol_info_tick(symbol)` to retrieve the latest `bid`, `ask`, and calculates `spread_pips` using the derived pip size.
5.  **Data Normalization**:
    - Converts the raw rate arrays into **Pandas DataFrames** with columns: `time`, `open`, `high`, `low`, `close`, `tick_volume`.
    - Ensures timestamps are timezone-aware.
6.  **Validation**:
    - Verifies that the fetched DataFrames are not empty and contain at least the minimum required bars (e.g., 60 bars for M1 to calculate reliable indicators). If insufficient, it returns an error state requesting a data refresh.

---

# Outputs

Returns a structured JSON object containing the raw data ready for `feature-engineering`.

```json
{
  "metadata": {
    "symbol": "XAUUSD!",
    "cross_symbol": "EURUSD!",
    "fetched_at": "2026-07-03T14:30:00Z",
    "pip_size": 0.1,
    "bid": 4124.80,
    "ask": 4125.06,
    "spread_pips": 0.26
  },
  "gold_data": {
    "df_m1": "<pandas.DataFrame shape=(200,6)>",
    "df_m5": "<pandas.DataFrame shape=(120,6)>",
    "df_m15": "<pandas.DataFrame shape=(80,6)>"
  },
  "cross_data": {
    "df_m1": "<pandas.DataFrame shape=(200,6)>",
    "df_m5": "<pandas.DataFrame shape=(120,6)>",
    "df_m15": "<pandas.DataFrame shape=(80,6)>"
  },
  "errors": []
}