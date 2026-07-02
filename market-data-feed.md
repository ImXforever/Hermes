---
name: market-data-feed
description: Fetches real-time and historical market data for 35+ trading symbols (including XAUUSD!, BTCUSD, indices, and cryptos) from MT5 or other bridges. Returns normalized OHLCV, bid/ask, and daily change.
version: 1.1.0
---

# Purpose

Provides a unified, clean, and timestamped data stream for any instrument supported by the trading platform. Acts as the single source of truth for all market-dependent skills. This version is pre-validated against the full symbol list from the user's MT5 terminal.

---

# Supported Symbols

The skill has been validated with the following instruments (exact string matching is handled internally):

**Precious Metals:**
`XAGAUD!`, `XAGUSD!`, `XAUAUD!`, `XAUEUR!`, `XAUUSD!`, `XPDUSD!`, `XPTUSD!`

**Indices:**
`ASXAUD!`, `DAXEUR!`, `ESXEUR!`, `F40EUR!`, `FTSGBP!`, `HSIHKD!`, `IBXEUR!`, `DJIUSD!`, `NDXUSD!`, `SPXUSD!`, `NIKJPY!`, `XINUSD!`

**Cryptocurrencies (no ! suffix):**
`BCHUSD`, `BTCUSD`, `ETHUSD`, `LTCUSD`, `SOLUSD`, `AAVUSD`, `ADAUSD`, `ALGUSD`, `AVAUSD`, `BATUSD`, `DOGUSD`, `DOTUSD`, `LNKUSD`, `SHBUSD`, `TDVIUSD`

---

# Inputs

The skill expects the following parameters when invoked:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbol` | string | Yes | One of the supported symbols (e.g., XAUUSD!, BTCUSD). Case-sensitive. |
| `timeframe` | string | Yes | Candle interval (e.g., M1, M5, M15, H1, H4, D1) |
| `bars_count` | integer | Yes | Number of candles/bars to fetch (min: 5, max: 500) |
| `include_bid_ask` | boolean | No | If true, returns current bid/ask and daily change alongside candles (default: true) |
| `include_volume` | boolean | No | Whether to include tick/contract volume (default: true) |
| `include_spread` | boolean | No | Whether to include spread data if available (default: false) |

---

# Execution

1. **Symbol Validation**: The skill first checks the `symbol` against an internal allowlist (see Supported Symbols). If not found, it returns a clear error with the closest match suggestion.
2. **Connection**: Establishes a secure connection to the configured data bridge (MT5 MCP Bridge, REST API, or WebSocket).
3. **Query**: Sends the request with the provided `symbol`, `timeframe`, and `bars_count`.
4. **Bid/Ask Snapshot**: If `include_bid_ask` is true, fetches the latest bid, ask, and daily change percentage.
5. **Validation**:
   - Verifies the response structure.
   - Checks for missing values, gaps in time, or malformed fields.
   - Ensures timestamp drift is within the acceptable threshold.
6. **Normalization**: Converts the raw data into a standardized JSON schema (see Outputs). For symbols with `!`, the suffix is stripped internally to match the bridge's expected format (MT5 handles `!` natively, so this is transparent).
7. **Cache**: Optionally caches the result for the defined TTL.
8. **Delivery**: Passes the structured data object to the next skill.

---

# Outputs

Returns a structured JSON object with the following schema:

```json
{
  "metadata": {
    "symbol": "XAUUSD!",
    "timeframe": "M5",
    "fetched_at": "2026-07-03T14:30:00Z",
    "source": "mt5",
    "bars_count": 100,
    "bid": 4124.80,
    "ask": 4125.06,
    "daily_change_percent": 0.04
  },
  "candles": [
    {
      "time": "2026-07-03T14:25:00Z",
      "open": 4124.50,
      "high": 4125.20,
      "low": 4123.90,
      "close": 4124.80,
      "tick_volume": 452,
      "spread": 0.26
    }
  ],
  "errors": []
}