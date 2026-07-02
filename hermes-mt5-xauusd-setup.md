# Hermes Agent × MetaTrader 5 — Automated XAUUSD Trading Setup

**Goal:** Connect Hermes Agent to your MT5 terminal, run an automated buy/sell decision every 1 minute on XAUUSD, log every trade outcome, and let Hermes turn that history into a self-improving skill that refines its own strategy code over time.

> ⚠️ **Risk Notice**: XAUUSD (gold) on a 1-minute timeframe is extremely volatile and spread-sensitive. An LLM-driven agent making automatic entries every 60 seconds can lose money very fast, especially with real funds. Start on a **demo account** only, run it for at least 1–2 weeks, and review every trade log before ever considering a live account. This guide is a technical setup walkthrough, not trading advice.

---

## 1. Prerequisites

| Requirement | Detail |
|---|---|
| OS | Windows (MT5's official Python API only runs on Windows) |
| MT5 Terminal | Installed and logged into a **demo** account |
| MT5 Setting | `Tools → Options → Expert Advisors → Allow algorithmic trading` ✅ |
| Python | 3.10+ (Hermes installer already brings its own, but MT5 bridge needs a Windows-native Python) |
| Hermes Agent | Already installed (confirmed from your screenshot) |

---

## 2. Install the MT5 MCP Bridge

Open a terminal (PowerShell) and run:

```bash
pip install metatrader-mcp-server
```

Test that it can see your terminal:

```bash
metatrader-mcp-server --login YOUR_LOGIN --password YOUR_PASSWORD --server YOUR_BROKER_SERVER
```

If it starts without errors, the bridge works. Stop it with `Ctrl+C` — Hermes will launch it itself.

---

## 3. Register the MCP Server Inside Hermes

In the Hermes dashboard (the panel in your screenshot):

1. Go to **MCP** in the left sidebar.
2. Click **Add MCP Server**.
3. Choose **stdio / local command** (not URL).
4. Fill in:

```
Name: mt5-xauusd
Command: metatrader-mcp-server
Args: --login YOUR_LOGIN --password YOUR_PASSWORD --server YOUR_BROKER_SERVER
```

5. Save, then restart the gateway (**Restart Gateway** button, bottom-left of your dashboard).

Verify the connection by typing in **Chat**:

```
Initialize MT5 and show me my account balance, equity, and margin.
```

Hermes should respond with real numbers from your demo account.

---

## 4. Create a Dedicated Profile (recommended)

Isolating this bot in its own profile keeps it from mixing memory/skills with your general assistant.

Go to **Profiles → New Profile**, name it e.g. `gold-scalper`, and select the `mt5-xauusd` MCP server for this profile only.

---

## 5. Seed the Trading Skill (first prompt to Hermes)

Paste this into Chat inside the `gold-scalper` profile. This is the **founding prompt** — Hermes will use it to write the first version of the trading skill.

```
I want you to build and maintain a self-improving XAUUSD trading skill called
"gold-scalper-v1". Requirements:

1. Every time you're triggered, fetch the last 100 M1 candles for XAUUSD using
   copy_rates_from_pos, plus the current bid/ask tick.
2. Compute a simple technical signal (start with EMA(9)/EMA(21) crossover +
   RSI(14) filter) to decide BUY, SELL, or NO TRADE.
3. If a signal fires and there is no open position on XAUUSD, open a position
   with 0.01 lots, a stop loss of 150 points, and a take profit of 300 points,
   using order_send.
4. If a position is already open, check whether it should be closed based on
   the same logic, or left running.
5. After every action (trade opened, trade closed, or no trade), log to a
   local file gold-scalper-log.jsonl: timestamp, signal, action taken, price,
   SL/TP, and current equity.
6. Turn this entire procedure into a reusable Skill (SKILL.md) named
   gold-scalper-v1 so it persists across sessions and cron runs.

Do not execute any trade yet — just build the skill, dry-run it once in
simulation mode (no order_send), and show me the decision it would have made.
```

Review the dry run output before continuing.

---

## 6. Schedule It to Run Every Minute (Cron)

Go to **CRON** in the sidebar → **New Job**.

```
Name: gold-scalper-tick
Schedule: * * * * *   (every minute)
Profile: gold-scalper
Prompt: Run the gold-scalper-v1 skill now. Execute real trades only if
        Simulation Mode is OFF. Log the outcome as usual.
Delivery: (optional) send a summary to Telegram/Discord if configured
```

Keep a **"Simulation Mode: ON"** flag inside the skill's config at first (ask Hermes to add this toggle) so it logs decisions without placing real orders until you're confident.

---

## 7. Turning On the Self-Improvement Loop

This is the part that makes it "get smarter." Hermes already has a built-in learning loop, but you need to explicitly ask it to close the feedback cycle. Use this prompt periodically (e.g., once a day, or set as its own daily cron job):

```
Review gold-scalper-log.jsonl from the last 24 hours. For every closed trade,
compute win rate, average R multiple, and drawdown. Identify which signal
conditions led to losing trades. Propose a specific, testable change to the
gold-scalper-v1 skill's entry/exit logic (e.g., adjusting RSI thresholds,
adding a volatility filter, changing SL/TP ratio). Update the skill's code
accordingly, bump its version to gold-scalper-v2, and explain exactly what
changed and why, citing the trade data that justified it. Keep the previous
version available in case the new one underperforms.
```

Optional but recommended — use Hermes's own evolutionary tooling instead of manual prompting:

```bash
git clone https://github.com/NousResearch/hermes-agent-self-evolution.git
cd hermes-agent-self-evolution
pip install -e ".[dev]"
export HERMES_AGENT_REPO=~/.hermes/hermes-agent

python -m evolution.skills.evolve_skill \
  --skill gold-scalper-v1 \
  --iterations 10 \
  --eval-source sessiondb
```

This runs a GEPA-based optimizer that reads your actual session/trade traces and proposes evidence-backed improvements to the skill automatically, rather than relying on the model's own self-assessment each day.

---

## 8. Guardrails Worth Adding (ask Hermes to build these in)

```
Add the following safety rails to gold-scalper-v1:
1. A daily max-loss limit (e.g., stop trading for 24h if equity drops 3% from
   day-start).
2. A max of 1 open XAUUSD position at any time.
3. A cooldown of 5 minutes after any closed trade before opening a new one.
4. A "circuit breaker" that pauses trading and messages me on Telegram if
   3 consecutive trades are losses.
```

---

## 9. Suggested Rollout Order

1. Run in **Simulation Mode** for several days — check `gold-scalper-log.jsonl` daily.
2. Turn on real order execution on the **demo account** only.
3. Let the daily self-improvement prompt (or the evolution script) run for 1–2 weeks.
4. Only after a consistent, reviewed track record would you consider a live account — and even then, start with minimum lot size.

---

## Quick Reference — Prompts Recap

| Step | Prompt file location in this doc |
|---|---|
| Build the skill | Section 5 |
| Run every minute | Section 6 (cron config, not a chat prompt) |
| Self-improve daily | Section 7 |
| Add safety limits | Section 8 |
