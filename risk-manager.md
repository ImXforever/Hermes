---
name: risk-manager
description: Enforces strict pre-trade risk filters (spread, ATR, session, news, layer distance, confidence) and computes Kelly-based position sizing for the GOLD SNIPER v9.0 agent. Implements the ScalpRiskController logic including daily drawdown kill switch and emergency position closure.
version: 1.0.0
---

# Purpose

Acts as the **gatekeeper** of the trading pipeline. Before any trade signal is executed, this skill validates that all market conditions are optimal for scalping. It ensures the agent does not trade during high volatility (news), low volatility (dead market), outside London/NY sessions, or when spread is excessive. It also enforces the maximum number of concurrent positions (layers) and prevents stacking orders too close to each other. If all checks pass, it calculates the optimal lot size using the Kelly Criterion (scaled by ATR) and returns a fully validated order bundle ready for execution.

---

# Inputs

The skill expects the following parameters, matching the `validate` and `kelly_lot` methods:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | Yes | The trading instrument (e.g., `XAUUSD!`). |
| `action_code` | integer | Yes | Proposed action: `0` (HOLD), `1` (BUY), `2` (SELL). |
| `current_bid` | float | Yes | Current bid price for spread calculation. |
| `current_ask` | float | Yes | Current ask price for spread calculation. |
| `spread_pips` | float | Yes | Current spread in pips (computed by `market-data-feed`). |
| `atr_pips` | float | Yes | Current ATR in pips (computed by `feature-engineering`). |
| `confidence` | float | Yes | The final confidence score from `rl-policy-decision`. |
| `open_positions` | list | Yes | List of currently open positions for this symbol (each with `price_open` and `type`). |
| `current_time_utc` | datetime | Yes | Current UTC time for session/news validation. |
| `win_rate` | float | No | Historical win rate (default: `0.5`). |
| `avg_win_pips` | float | No | Average win in pips (default: `TP1_PIPS = 15`). |
| `avg_loss_pips` | float | No | Average loss in pips (default: `SL_PIPS_MIN = 8`). |
| `equity` | float | Yes | Current account equity for daily drawdown calculation. |
| `balance` | float | Yes | Current account balance (used as starting equity reference). |

---

# Execution

1.  **Action Filter**:
    - If `action_code` is `0` (HOLD), the skill immediately returns a `valid: false` with reason "HOLD action".

2.  **Spread Filter**:
    - Compares `spread_pips` against `MAX_SPREAD_PIPS` (default: `3.5` pips).
    - If spread exceeds the limit, validation fails with reason "Spread too high".

3.  **ATR (Volatility) Filter**:
    - If `atr_pips` > `MAX_ATR_PIPS` (default: `180`): Fails with reason "ATR too high (news/volatility)".
    - If `atr_pips` < `MIN_ATR_PIPS` (default: `8`): Fails with reason "ATR too low (dead market)".

4.  **Session Filter**:
    - Checks the `current_time_utc` against the configured London and NY trading hours.
    - Trading is allowed only between `LONDON_OPEN` (7:00 UTC) and `LONDON_CLOSE` (16:00 UTC) OR `NY_OPEN` (12:00 UTC) and `NY_CLOSE` (20:00 UTC).
    - Outside these hours, validation fails with reason "Outside trading session".

5.  **News Blackout Filter**:
    - Checks if `current_time_utc` is within `NEWS_BLACKOUT_MIN` (default: `30` minutes) of any event in the news cache.
    - If inside a blackout, validation fails with reason "News blackout active".

6.  **Position Layer Filter**:
    - Counts the number of currently open positions for the symbol.
    - If count >= `MAX_LAYERS` (default: `2`), validation fails with reason "Max layers reached".
    - Checks the distance (in pips) between the current market price and the open price of each existing position.
    - If the distance is less than `MIN_LAYER_DISTANCE` (default: `12` pips), validation fails with reason "Too close to existing layer".

7.  **Confidence Filter**:
    - If `confidence` < `MIN_CONFIDENCE` (default: `0.62`), validation fails with reason "Confidence below threshold".

8.  **Position Sizing (Kelly Criterion)**:
    - **Payoff Ratio**: `payoff_ratio = avg_win_pips / avg_loss_pips`.
    - **Kelly Fraction**: `kelly = win_rate - (1 - win_rate) / payoff_ratio`. Clamped between `0.0` and `1.0`.
    - **Scaled Kelly**: `scaled_kelly = kelly * KELLY_FRACTION` (default `0.25`).
    - **ATR Scale Factor**: `atr_scale = min(1.0, 20.0 / atr_pips)` — reduces lot size in high volatility.
    - **Final Lot**: `lot = VOLUME_BASE * (1 + scaled_kelly) * atr_scale`.
    - **Clamping**: Rounds to 3 decimals and clamps between `MIN_LOT_SIZE` (0.001) and `MAX_LOT_SIZE` (0.1).
    - If `lot` is less than the symbol's minimum contract size, it rounds up to the minimum.

9.  **Equity Monitoring (Daily Drawdown)**:
    - The skill maintains an internal state for `daily_high` (peak equity for the day).
    - On every call, it updates `daily_high = max(daily_high, equity)`.
    - Computes current drawdown: `dd = (daily_high - equity) / daily_high`.
    - If `dd >= MAX_DAILY_DD_PCT` (default: `0.04` = 4%), the skill activates the **Kill Switch**:
        - Sets `kill_switch = True`.
        - Logs a critical event.
        - Triggers an `emergency_close` call (delegated to the `trade-executor` skill).
        - Returns validation failure with reason "Daily drawdown limit breached".

---

# Outputs

Returns a JSON object confirming the validation status and the calculated lot size.

```json
{
  "symbol": "XAUUSD!",
  "timestamp": "2026-07-03T14:30:00Z",
  "valid": true,
  "reason": "All filters passed",
  "lot": 0.02,
  "kill_switch_active": false,
  "metadata": {
    "spread_pips": 0.8,
    "atr_pips": 15.2,
    "current_layers": 1,
    "closest_position_pips": 25.0,
    "confidence": 0.78,
    "kelly_raw": 0.15,
    "atr_scale": 0.95
  },
  "errors": []
}