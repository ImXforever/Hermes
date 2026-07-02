---
name: rl-policy-decision
description: Selects optimal trading actions (BUY/SELL/HOLD) for GOLD SNIPER v9.0 by fusing FusedActorCritic RL inference with ICT market structure signals (Order Blocks, FVGs, Liquidity Sweeps). Applies Kelly-based position sizing and generates precise SL/TP levels compatible with MT5.
version: 1.0.0
---

# Purpose

Acts as the **decision-making core** of the GQRAgent. It consumes the 28-dimensional feature vectors for both Gold and EURUSD, runs them through the trained `FusedActorCritic` neural network, and combines the result with real-time ICT structure detection. This hybrid approach ensures that when the RL model is uncertain (confidence below `MIN_CONFIDENCE`), the agent can still trade based on high-probability technical setups (sweeps, order blocks, FVGs). The skill outputs a complete trading signal including volume, entry price, stop-loss, and take-profit levels.

---

# Inputs

The skill expects the following parameters, matching the `run_live_step` context:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `gold_feature_vector` | array (28 floats) | Yes | The normalized Gold feature vector from `feature-engineering`. |
| `cross_feature_vector` | array (28 floats) | Yes | The normalized EURUSD feature vector from `feature-engineering`. |
| `current_bid` | float | Yes | Current bid price of the symbol. |
| `current_ask` | float | Yes | Current ask price of the symbol. |
| `available_balance` | float | Yes | Account equity or free margin for Kelly calculation. |
| `open_positions_count` | integer | Yes | Number of currently open positions for this symbol. |
| `closest_position_distance_pips` | float | Yes | Distance (in pips) from the current price to the nearest open position (to enforce `MIN_LAYER_DISTANCE`). |
| `structure_data` | dict | Yes | The output from `compute_structure_features` (contains bull/bear OB, FVG, sweep flags). |
| `symbol_pip` | float | Yes | Broker-specific pip size for the symbol (from `PipInfo`). |
| `win_rate` | float | No | Historical win rate (default: 0.5). |
| `avg_win_pips` | float | No | Average winning trade in pips (default: TP1_PIPS = 15). |
| `avg_loss_pips` | float | No | Average losing trade in pips (default: SL_PIPS_MIN = 8). |

---

# Execution

1.  **RL Model Inference**:
    - Converts the input feature vectors into PyTorch tensors.
    - Performs a forward pass through the `FusedActorCritic` model (in evaluation mode).
    - Obtains action logits and applies softmax to derive action probabilities.
    - Extracts the `rl_action` (argmax of probabilities) and `rl_confidence` (max probability).

2.  **ICT Structure Signal Extraction**:
    - Parses the `structure_data` to detect:
      - **Bullish signals**: `bull_sweep` combined with `bull_ob`.
      - **Bearish signals**: `bear_sweep` combined with `bear_ob`.
    - Calculates `structure_signal` (1 = BUY, 2 = SELL, 0 = HOLD).
    - Calculates `structure_confidence` (scaled based on OB score and recency).
    - If a structure signal is present, sets `structure_entry_price` (mid of OB for market orders, or specific entry logic) and `sl_anchor` (OB low for BUY, OB high for SELL).

3.  **Fusion Decision Logic** (Exact match to v9.0 code):
    - **Case A**: If `rl_confidence` >= `MIN_CONFIDENCE` → Use `rl_action` as the `final_action` and `rl_confidence` as `final_confidence`. Use structure data only to enhance the SL/TP anchor (if available).
    - **Case B**: If `rl_confidence` < `MIN_CONFIDENCE` AND `structure_confidence` >= `MIN_CONFIDENCE` → Override! Use `structure_signal` as `final_action` and set `final_confidence` to `structure_confidence`. This ensures the agent doesn't miss high-probability manual setups when the AI is uncertain.
    - **Case C**: If both are below threshold → Force `HOLD` (action 0).

4.  **Entry Price & SL/TP Construction**:
    - If a structure (Case B) or structure price exists: Sets a limit order entry using `entry_price` from the OB/mid.
    - Sets the `sl_price`:
      - If `sl_anchor` exists, calculates SL pips as distance from entry to anchor (bounded by `SL_PIPS_MIN` and `SL_PIPS_MAX`).
      - Otherwise, defaults to `SL_PIPS_MIN`.
    - Sets `tp1_price` (based on `TP1_PIPS`) and `tp2_price` (based on `TP2_PIPS`).

5.  **Position Sizing (Kelly Criterion)**:
    - Uses the formula from the v9.0 code: `kelly = win_rate - (1 - win_rate) / payoff_ratio`.
    - Multiplies by `KELLY_FRACTION` (0.25) to ensure conservative sizing.
    - Scales by `atr_scale` (penalizes trading in high volatility).
    - Outputs the final `lot` (rounded to 3 decimals, clamped between 0.001 and 0.1).

6.  **Pre-Execution Validation** (Soft checks):
    - Ensures `final_action` is not 0.
    - Verifies that `open_positions_count` is less than `MAX_LAYERS`.
    - Verifies `closest_position_distance_pips` is greater than `MIN_LAYER_DISTANCE`.
    - Verifies the confidence is not NaN and is within the range [0, 1].

---

# Outputs

Returns a JSON object containing the fully prepared trading signal for the `trade-executor` skill.

```json
{
  "symbol": "XAUUSD!",
  "timestamp": "2026-07-03T14:30:00Z",
  "action_code": 1,
  "action_label": "BUY",
  "confidence": 0.78,
  "decision_source": "rl_model",  // or "structure_override"
  "lot": 0.02,
  "entry_price": 4125.00,
  "stop_loss_price": 4120.00,
  "take_profit_price": 4140.00,
  "tp2_price": 4153.00,
  "metadata": {
    "rl_confidence": 0.72,
    "structure_confidence": 0.85,
    "structure_used": {
      "ob_high": 4124.50,
      "ob_low": 4120.00,
      "score": 0.85
    },
    "kelly_fraction": 0.25,
    "atr_scale_factor": 0.85
  },
  "errors": []
}