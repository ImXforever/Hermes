---
name: reward-calculator
description: Computes the RL reward signal for the GOLD SNIPER v9.0 agent. Supports live trading (equity percentage change) and simulation (detailed PnL, ATR, trade count, and Sharpe penalties). Directly mirrors the logic from GoldEnvFusion.step and GQRAgent.run_live_step.
version: 1.0.0
---

# Purpose

Acts as the **reward generation engine** for the reinforcement learning pipeline. It translates raw market outcomes (PnL, equity changes, holding duration) into a scalar reward value that guides the `FusedActorCritic` policy updates. The skill implements two distinct modes:
- **Live Mode**: Simple percentage change in account equity (stable and directly correlated with real profit/loss).
- **Simulation Mode**: Detailed reward shaping based on open/close actions, unrealized PnL thresholds (ATR-based), trade count penalties, and Sharpe ratio bonuses to accelerate learning in the `GoldEnvFusion` gym environment.

---

# Inputs

The skill expects a combination of market and position parameters, depending on the `mode` selected.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `mode` | string | Yes | `live` or `simulation` (must match the execution context). |
| `action_code` | integer | Yes | The action taken (`0`=HOLD, `1`=BUY, `2`=SELL, `3`=CLOSE). |
| `current_price` | float | Yes | Current close/bid price for PnL calculations. |
| `atr_pips` | float | Yes | Current ATR value (in pips) for threshold-based rewards. |
| `pip_size` | float | Yes | Broker-specific pip value. |

**Live Mode Inputs:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `equity_before` | float | Yes | Account equity before the trade/step. |
| `equity_after` | float | Yes | Account equity after the trade/step. |

**Simulation Mode Inputs:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entry_price` | float | No | Price at which the position was opened (if `in_position`). |
| `position_direction` | string | No | `BUY` or `SELL` if `in_position`. |
| `in_position` | boolean | Yes | Whether a position is currently held. |
| `trade_count` | integer | Yes | Number of trades taken in the current episode. |
| `rewards_history` | list | No | Historical rewards list (for Sharpe ratio computation). |

---

# Execution

**Mode Selection**:
- If `mode == "live"`: The skill immediately computes the reward as the percentage change in equity.

**Live Mode Logic (from `GQRAgent.run_live_step`)**:
1.  `reward = (equity_after - equity_before) / (abs(equity_before) + 1e-8)`
2.  This value is returned directly (range is typically around ±0.1% to ±5% per step).

**Simulation Mode Logic (from `GoldEnvFusion.step`)**:

1.  **Opening a Position (`action == 1` or `2`)**:
    - `reward = -0.03` (small penalty for opening, to discourage excessive trading).
    - Position state is updated (entry price, direction, in_position = True).

2.  **Closing a Position (`action == 3`)**:
    - Calculates raw PnL in pips: `raw_pnl_pips = ((current_price - entry_price) if direction == "BUY" else (entry_price - current_price)) / pip_size`.
    - **Reward Asymmetry** (Loss Aversion):
      - If `raw_pnl_pips > 0` (Profitable): `reward = raw_pnl_pips * 1.0`
      - If `raw_pnl_pips <= 0` (Losing): `reward = raw_pnl_pips * 1.5` (penalizes losses 50% more to encourage risk-averse behavior).
    - Resets position state (`in_position = False`).

3.  **Holding a Position (in_position and `action == 0` or any non-close action)**:
    - Calculates current unrealized PnL in pips: `raw_pnl_pips` (same formula).
    - **ATR-Based Stop-Loss Guard (1.5x ATR)**:
      - If `raw_pnl_pips <= -(atr_pips * 1.5)`: Forces a loss exit.
        - `reward = raw_pnl_pips * 1.5`
        - `in_position = False` (position auto-closes).
    - **ATR-Based Take-Profit Guard (3x ATR)**:
      - If `raw_pnl_pips >= (atr_pips * 3.0)`: Forces a profit exit.
        - `reward = raw_pnl_pips * 1.0`
        - `in_position = False` (position auto-closes).
    - **Drift Reward**:
      - If neither threshold is breached: `reward = raw_pnl_pips * 0.05` (small positive/negative drift signal to encourage favorable price movement).

4.  **Trade Count Penalty (Prevents Overtrading)**:
    - If `trade_count > 15`: `reward -= 0.05 * (trade_count - 15)`
    - This progressively penalizes the agent for taking too many trades in an episode, encouraging quality over quantity.

5.  **Sharpe Ratio Bonus (Rewards Consistency)**:
    - If `len(rewards_history) >= 20`:
      - Calculates the Sharpe ratio of the recent rewards: `sharpe = mean(rewards_history[-20:]) / (std(rewards_history[-20:]) + 1e-9)`.
      - `reward += sharpe * 0.02` (adds a small bonus for consistent positive returns).

6.  **Episode Termination Handling**:
    - When the episode ends (`terminated == True`):
      - If `in_position` is `True`, it forces a close: `reward += raw_pnl_pips * (1.0 if raw_pnl_pips > 0 else 1.5)`.
      - This ensures the agent is penalized/rewarded for any open position at the end of the data window.

---

# Outputs

Returns a JSON object containing the calculated reward and detailed metadata explaining the components.

```json
{
  "timestamp": "2026-07-03T14:30:00Z",
  "reward": 0.015,
  "mode": "live",
  "metadata": {
    "equity_before": 10000.0,
    "equity_after": 10150.0,
    "percent_change": 0.015
  },
  "components": {
    "pnl_impact": 0.015,
    "trade_count_penalty": 0.0,
    "sharpe_bonus": 0.0
  },
  "errors": []
}