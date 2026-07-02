---
name: trade-executor
description: Executes market and pending limit orders on MetaTrader 5 for the GOLD SNIPER v9.0 agent. Implements the MT5TradeExecutor logic including fill-mode detection, SL/TP construction, emergency close, and handling of trade deviations.
version: 1.0.0
---

# Purpose

Acts as the **final execution layer** of the trading pipeline. It receives the structured trading signal (action, volume, entry price, stop-loss, take-profit) from the `rl-policy-decision` skill, translates it into an MT5-compatible order request, and submits it to the broker. This skill handles the critical differences between market orders and limit orders, manages partial fills, and ensures that all trades are stamped with the correct `MAGIC` number for identification.

---

# Inputs

The skill expects the following parameters, which map directly to the output of `rl-policy-decision`:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | Yes | The trading symbol (e.g., `XAUUSD!`). |
| `action_code` | integer | Yes | `0` = HOLD (skip), `1` = BUY, `2` = SELL. |
| `lot` | float | Yes | The calculated trade volume (from Kelly sizing). |
| `entry_price` | float | No | If provided, a **PENDING (limit)** order is placed at this price. If `None` or `0`, a **MARKET** order is placed using the current bid/ask. |
| `stop_loss_price` | float | Yes | The stop-loss price level. |
| `take_profit_price` | float | Yes | The primary take-profit price level (TP1). |
| `take_profit_2_price` | float | No | The secondary take-profit price level (TP2) — used for partial close management by the `position-manager` skill, not directly in this order. |
| `magic_number` | integer | No | The unique identifier for the EA (default: `999111` from v9.0). |
| `deviation` | integer | No | Allowed slippage in points (default: `20`). |
| `comment` | string | No | Order comment for identification (default: `GQR_Scalper_v9`). |

---

# Execution

1.  **Action Validation**:
    - If `action_code` is `0` (HOLD), the skill immediately returns a success state without sending any order.
    - If `lot` is less than `0.001` or greater than `100`, the skill clamps the value and logs a warning.

2.  **Market Price Fetch**:
    - Calls `mt5.symbol_info_tick(symbol)` to retrieve the current `bid` and `ask`.
    - Determines the order type:
        - For **BUY**: Market price = `ask`, Limit price = `entry_price` (if provided).
        - For **SELL**: Market price = `bid`, Limit price = `entry_price` (if provided).

3.  **Order Type Decision** (Exact logic from `_execute_sync`):
    - **Condition**: If `entry_price` is provided AND the absolute difference between `entry_price` and `market_price` is greater than `1e-9` (i.e., meaningful difference):
        - **Use PENDING (Limit) order**: `ORDER_TYPE_BUY_LIMIT` or `ORDER_TYPE_SELL_LIMIT`.
        - Action = `TRADE_ACTION_PENDING`.
    - **Otherwise**:
        - **Use MARKET (Deal) order**: `ORDER_TYPE_BUY` or `ORDER_TYPE_SELL`.
        - Action = `TRADE_ACTION_DEAL`.

4.  **Fill Mode Detection**:
    - Calls the `get_fill_mode(symbol)` function (mirroring the code logic) to determine the correct `type_filling`:
        - `ORDER_FILLING_FOK` (Fill or Kill)
        - `ORDER_FILLING_IOC` (Immediate or Cancel)
        - `ORDER_FILLING_RETURN` (Return)

5.  **Order Construction**:
    - Builds a Python dictionary matching the `mt5.order_send()` request format:
        ```python
        request = {
            "action": mt5.TRADE_ACTION_DEAL or TRADE_ACTION_PENDING,
            "symbol": symbol,
            "volume": lot,
            "type": order_type,
            "price": market_price or entry_price,
            "sl": stop_loss_price,
            "tp": take_profit_price,
            "deviation": deviation,
            "magic": magic_number,
            "comment": comment,
            "type_time": mt5.ORDER_TIME_GTC,
            "type_filling": fill_mode,
        }