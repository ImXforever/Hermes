---
name: trajectory-learner
description: Implements the complete online RL learning loop for the GOLD SNIPER v9.0 agent. Handles experience storage (SQLite), batch sampling, PPO model updates, ModelGuard (Sharpe-based rollback), and audit logging. Mirrors the combined logic of SQLiteExperienceVault, PPOTrainer, and ModelGuard from the v9.0 codebase.
version: 1.0.0
---

# Purpose

Acts as the **memory and learning engine** of the RL agent. It collects transition tuples (state, action, reward, next_state) from the live trading environment, stores them persistently in a SQLite database, and periodically triggers PPO updates to improve the `FusedActorCritic` policy. The skill also includes safety mechanisms to prevent catastrophic forgetting and performance degradation through automated model rollbacks based on realized Sharpe ratio.

---

# Inputs

The skill expects the following parameters, which map directly to the `PPOTrainer.update` and `SQLiteExperienceVault.commit` methods:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `gold_state` | tensor/array | Yes | The 28-dim feature vector for Gold before action. |
| `cross_state` | tensor/array | Yes | The 28-dim feature vector for EURUSD before action. |
| `action` | integer | Yes | The action taken (`0`=HOLD, `1`=BUY, `2`=SELL). |
| `reward` | float | Yes | The computed reward from `reward-calculator`. |
| `next_gold_state` | tensor/array | Yes | The 28-dim feature vector for Gold after action. |
| `next_cross_state` | tensor/array | Yes | The 28-dim feature vector for EURUSD after action. |
| `log_prob` | float | Yes | Log probability of the selected action from the policy. |
| `value` | float | Yes | Estimated value of the state from the critic (V(s)). |
| `done` | boolean | Yes | Whether the episode has terminated (or buffer flush trigger). |
| `symbol` | string | No | The instrument (default: `XAUUSD!`). |
| `cross_symbol` | string | No | The cross instrument (default: `EURUSD!`). |

---

# Execution

1.  **Experience Storage (Commit)**:
    - The skill receives the experience tuple from the agent's live step.
    - It converts the `gold_state`, `cross_state`, `next_gold_state`, and `next_cross_state` to flat numpy arrays (byte strings for SQLite BLOB storage).
    - It appends the tuple to an in-memory buffer (mirroring the `_buffer` in `SQLiteExperienceVault`).
    - When the buffer reaches `flush_every` (default: `500` experiences), it commits all buffered experiences to the SQLite database in a single transaction (using `executemany`).

2.  **PPO Buffer Management**:
    - Simultaneously, the skill maintains a separate in-memory `buffer` for the PPO trainer (list of dicts containing `gold`, `cross`, `action`, `log_prob`, `reward`, `next_gold`, `next_cross`, `value`, `done`).
    - On every `commit` call, the skill also appends the full transition to this PPO buffer.

3.  **Training Trigger (`train_if_ready`)**:
    - The skill tracks the total number of steps stored in the PPO buffer.
    - When the buffer size reaches `TRAIN_STEPS` (default: `1024`), the skill automatically triggers a PPO update.

4.  **PPO Update (Exact Logic from `PPOTrainer.update`)**:
    - **Data Preparation**: Stacks all `gold`, `cross`, `next_gold`, `next_cross` tensors, and converts actions, log_probs, rewards, and values to tensors on the correct device.
    - **Advantage Calculation (GAE)**:
      - Computes the next-state values (V(s')) via a forward pass of the model.
      - Computes Temporal Difference (TD) errors: `delta = reward + gamma * V(s') - V(s)`.
      - Computes Generalized Advantage Estimation (GAE) using `gae_lambda`.
      - Computes returns: `returns = advantages + values`.
      - Normalizes advantages.
    - **Mini-Batch Updates**:
      - For `PPO_EPOCHS` (default: `5`):
        - Shuffles the data.
        - Iterates over mini-batches of size `PPO_BATCH_SIZE` (default: `256`).
        - For each batch:
          - Calculates new action logits and values via the model.
          - Computes policy ratio: `ratio = exp(new_log_prob - old_log_prob)`.
          - Computes clipped surrogate objective: `policy_loss = -min(ratio * advantage, clip(ratio, 1-clip, 1+clip) * advantage).mean()`.
          - Computes value loss (MSE between predicted values and returns).
          - Computes entropy bonus (to encourage exploration).
          - **Total Loss**: `loss = policy_loss + value_coeff * value_loss - entropy_coeff * entropy`.
          - Performs backpropagation and optimizer step (AdamW with weight decay).
    - **Returns**: The average loss over the last epoch.

5.  **ModelGuard (Performance Monitoring)**:
    - Before the PPO update, the skill calls `ModelGuard.backup()` to save a copy of the current model weights.
    - After the PPO update, the skill adds the `reward` to the guard's return history (`ModelGuard.add_return(reward)`).
    - If the history contains at least `evaluation_steps` (default: `500` returns), it computes the annualized Sharpe ratio:
      - `sharpe = (mean(returns) / (std(returns) + 1e-8)) * sqrt(seconds_per_year / cycle_seconds)`.
    - If `sharpe < sharpe_threshold` (default: `-0.5`):
      - The guard triggers a **rollback**: loads the backup model weights into the current model.
      - Logs the rollback event to the audit log.
      - Resets the return history and disables monitoring until the next backup.

6.  **Audit Logging**:
    - Every significant event (experience commit, PPO update start/end, model rollback) is logged to `AUDIT_LOG` with:
      - Timestamp.
      - Event type.
      - Data (buffer size, loss, Sharpe ratio, etc.).
      - Hash chain linking each entry to the previous one (for tamper-proof verification).

7.  **Buffer Cleanup**:
    - After a successful PPO update, the in-memory PPO buffer is cleared.
    - The SQLite database is not cleared; it accumulates experiences indefinitely (for future offline training).

---

# Outputs

The skill does not produce a direct output to the trading pipeline, but returns a status JSON for monitoring.

```json
{
  "timestamp": "2026-07-03T14:30:00Z",
  "status": "training_completed",
  "buffer_size_before": 1024,
  "training_loss": 0.234,
  "model_rollback_triggered": false,
  "sharpe_last_check": 0.85,
  "memory_db_size": 15423,
  "errors": []
}