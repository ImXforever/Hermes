---
name: feature-engineering
description: Transforms raw MT5 OHLCV data (for Gold and EURUSD) into the exact 28-dimensional normalized feature vectors required by the GOLD SNIPER v9.0 FusedActorCritic RL model. Includes ICT Order Block, FVG, and Liquidity Sweep detectors.
version: 1.0.0
---

# Purpose

Acts as the **state constructor** for the RL agent. It consumes the DataFrames provided by `market-data-feed` and calculates a comprehensive set of technical indicators, market structure signals, and position-aware metrics. The output is a flat, normalized feature vector that serves as the direct input to the policy network (Actor) and value network (Critic). This skill strictly follows the logic defined in the `_extract_features_for_model` and `_extract_cross_features` methods of the `GQRAgent`.

---

# Inputs

The skill expects the following parameters, mirroring the `run_live_step` function:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `gold_df_m1` | DataFrame (Pandas) | Yes | Gold 1-minute OHLCV data (must include `open`, `high`, `low`, `close`, `tick_volume`). |
| `gold_df_m5` | DataFrame (Pandas) | Yes | Gold 5-minute OHLCV data. |
| `gold_df_m15` | DataFrame (Pandas) | No | Gold 15-minute OHLCV data (falls back to M5 if unavailable). |
| `cross_df_m1` | DataFrame (Pandas) | Yes | EURUSD 1-minute OHLCV data for cross-fusion. |
| `cross_df_m5` | DataFrame (Pandas) | Yes | EURUSD 5-minute OHLCV data. |
| `cross_df_m15` | DataFrame (Pandas) | No | EURUSD 15-minute OHLCV data (falls back to M5 if unavailable). |
| `current_position_pnl` | float | No | Current unrealized PnL in pips (used for state feature #11). |
| `in_position` | boolean | No | Whether the agent currently holds a position. |
| `position_direction` | string | No | `BUY` or `SELL` if `in_position` is true. |
| `trade_count` | integer | No | Number of trades taken in the current episode (used for feature #19). |

---

# Execution

1.  **Data Preprocessing**:
    - Ensures all DataFrames are sorted by time and reset their indices.
    - If `gold_df_m15` or `cross_df_m15` is missing, it falls back to the respective M5 DataFrame (as per the v9.0 code logic).

2.  **Indicator Calculation (Gold)**:
    - **Price Deviation**: Computes deviations from EMAs (21, 50) normalized by pip size.
    - **Momentum**: RSI (14-period).
    - **Trend**: MACD (12, 26, 9) and Signal line difference.
    - **Volatility**: ATR (14-period) and Bollinger Band position.
    - **Volume**: Volume ratio (current / SMA of volume over 20 periods).
    - **Oscillators**: Stochastic (14, 3).

3.  **ICT Market Structure Detection (Gold)**:
    - Executes `detect_order_blocks_v9` to find bullish/bearish Order Blocks with a score.
    - Executes `detect_fvg_v9` to find Fair Value Gaps.
    - Executes `detect_liquidity_sweep_v9` to identify swept highs/lows and reversion potential.
    - Calculates `buy_momentum` and `sell_momentum` scores based on proximity to these structures.

4.  **Cross-Asset Feature Extraction (EURUSD)**:
    - Extracts the same technical indicators (RSI, ATR, MACD, etc.) for EURUSD.
    - *Note*: ICT Structure features are **not** calculated for the cross asset (they are zero-padded to maintain the 28-dim shape).

5.  **Position & PnL Integration**:
    - Injects the current unrealized PnL (in pips) into the state.
    - Encodes the `in_position` flag and direction as binary features.
    - Applies a `trade_count` penalty feature to prevent over-trading.

6.  **Normalization & Clipping**:
    - All raw values are scaled to a range appropriate for neural network input (e.g., dividing by pip sizes, normalizing by 100, clipping extremes).
    - Extra structure features (OB scores, sweep flags) are appended to reach exactly **28 dimensions** (`OBS_DIM`).
    - All NaN/Inf values are replaced with `0.0`.

7.  **State Assembly**:
    - Returns two separate 28-dim vectors: one for **Gold** and one for **EURUSD**. These are passed as separate inputs to the `FusedActorCritic` model (which fuses them via cross-attention).

---

# Outputs

Returns a JSON object containing the processed feature vectors and a readable interpretation.

```json
{
  "symbol": "XAUUSD!",
  "cross_symbol": "EURUSD!",
  "timestamp": "2026-07-03T14:30:00Z",
  "gold_feature_vector": [
    -0.021,  // Feature 0: EMA21 deviation
    0.015,   // Feature 1: EMA5 deviation
    0.03,    // Feature 2: Spread
    1.374,   // Feature 3: Price normalized
    -0.001,  // Feature 4: EMA ratio
    0.154,   // Feature 5: RSI normalized
    0.003,   // Feature 6: MACD
    0.002,   // Feature 7: MACD Signal diff
    0.2,     // Feature 8: BB position
    1.25,    // Feature 9: Volume ratio
    0.5,     // Feature 10: Stochastic
    0.02,    // Feature 11: ATR pips / 100
    0.01,    // Feature 12: EMA50 deviation
    0.8,     // Feature 13: Buy momentum (MM)
    0.2,     // Feature 14: Sell momentum
    1.0,     // Feature 15: Bull Sweep flag
    0.0,     // Feature 16: Bear Sweep flag
    1.0,     // Feature 17: In BUY position
    0.0,     // Feature 18: In SELL position
    0.1,     // Feature 19: Trade count / 50
    0.3,     // Feature 20: EMA deviation / ATR
    0.6,     // Feature 21: RSI/100
    0.8,     // Feature 22: Volume ratio / 2
    0.5,     // Feature 23: BB position raw
    0.45,    // Feature 24: Bull OB score
    0.0,     // Feature 25: Bear OB score
    1.0,     // Feature 26: Bull Sweep (duplicate for structure)
    0.0      // Feature 27: Bear Sweep (duplicate for structure)
  ],
  "cross_feature_vector": [
    // 28-dim vector for EURUSD (structure features zero-padded)
    ... 
  ],
  "indicators_interpretation": {
    "gold_rsi": 65.4,
    "gold_atr_pips": 18.5,
    "gold_macd_hist": 0.85,
    "bullish_order_block": { "high": 4125.0, "low": 4120.0, "score": 0.45 },
    "bearish_order_block": null
  },
  "errors": []
}