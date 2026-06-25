# Alert Rules Reference

Rules are evaluated per-transaction for each watched contract.
Multiple rules can match the same transaction — each fires an independent webhook call.
A rule evaluation error is logged as a warning and skipped; it never stops the engine.

## Rule types

### `AnyTransaction`
Matches every transaction that appears in the contract's Horizon history.

**Use case:** full audit trail, low-volume contracts.

```toml
[[contracts.rules]]
type = "AnyTransaction"
```

### `TransactionFailed`
Matches transactions where `successful = false`.

**Use case:** detect reverted Soroban invocations or fee-bump failures.

```toml
[[contracts.rules]]
type = "TransactionFailed"
```

### `TransactionSucceeded`
Matches transactions where `successful = true`. Inverse of `TransactionFailed`.

**Use case:** track successful escrow release transactions or settlement confirmations.

```toml
[[contracts.rules]]
type = "TransactionSucceeded"
```

### `LargeTransfer`

| Field           | Type | Required | Description                        |
|-----------------|------|----------|------------------------------------|
| `threshold_xlm` | u64  | yes      | Minimum transfer amount in XLM (> 0) |

Matches when the payment amount (extracted from Horizon operations) is ≥ `threshold_xlm` XLM.
The `amount_xlm` field in the webhook payload contains the actual transferred amount.

**Note:** Amount is extracted from `payment` operation records. Soroban token transfers
that do not produce a native `payment` operation will not populate `amount_xlm`.

```toml
[[contracts.rules]]
type          = "LargeTransfer"
threshold_xlm = 10000
```

### `SmallTransfer`

| Field     | Type | Required | Description                        |
|-----------|------|----------|------------------------------------|
| `min_xlm` | u64  | yes      | Minimum transfer amount in XLM (> 0) |

Matches when the payment amount is **less than** `min_xlm` XLM (opposite of `LargeTransfer`).
Useful for detecting spam attacks with micro-transfers or monitoring for dust amounts.

**Use case:** detect spam attacks, micro-transaction fraud, or unusually small transfer activity.

```toml
[[contracts.rules]]
type    = "SmallTransfer"
min_xlm = 1
```

### `FunctionCalled`

| Field           | Type   | Required | Description                          |
|-----------------|--------|----------|--------------------------------------|
| `function_name` | string | yes      | Exact function name (case-sensitive) |

Matches when the Soroban `invoke_host_function` operation calls exactly `function_name`.

```toml
[[contracts.rules]]
type          = "FunctionCalled"
function_name = "withdraw"
```

### `MemoContains`

| Field        | Type   | Required | Description                              |
|--------------|--------|----------|------------------------------------------|
| `memo_text`  | string | yes      | Text to match against (case-sensitive)  |
| `match_mode` | string | no       | Match mode: `"exact"`, `"contains"`, `"starts_with"` (default: `"exact"`) |

Matches when the transaction has a text memo matching the specified text according to the chosen match mode.
Only applies to transactions with `memo_type == "text"`.

**Match modes:**
- `exact`: memo must equal `memo_text` exactly
- `contains`: memo must contain `memo_text` as a substring
- `starts_with`: memo must start with `memo_text`

**Use case:** detect escrow transactions with specific reference codes, payment batches with identifying markers, or transaction categorization by memo.

```toml
[[contracts.rules]]
type       = "MemoContains"
memo_text  = "refund"
match_mode = "contains"
```

### `AdminFunctionCalled`

| Field            | Type     | Required | Description                              |
|------------------|----------|----------|------------------------------------------|
| `function_names` | [string] | yes      | Non-empty list of function names to watch |

Matches when the invoked function is any entry in `function_names`.
Equivalent to multiple `FunctionCalled` rules but produces a single
`AdminFunctionCalled([...])` label in the alert.

```toml
[[contracts.rules]]
type           = "AdminFunctionCalled"
function_names = ["set_admin", "upgrade", "initialize"]
```

### `HighFee`

| Field                | Type | Required | Description                           |
|----------------------|------|----------|---------------------------------------|
| `threshold_stroops`  | u64  | yes      | Fee threshold in stroops (> 0)        |

Matches when the transaction's total fee exceeds `threshold_stroops`.
The `fee_charged` field in the webhook payload contains the actual fee paid in stroops.

**Note:** Stroops are the smallest unit of XLM (1 XLM = 10,000,000 stroops).

```toml
[[contracts.rules]]
type               = "HighFee"
threshold_stroops  = 100000
```

### `AlertCooldown` modifier

| Field              | Type   | Required | Description                              |
|--------------------|--------|----------|------------------------------------------|
| `cooldown_seconds` | u64    | no       | Minimum seconds between alerts for this rule (> 0 if set) |

The `cooldown_seconds` field can be added to any rule to suppress repeated alerts within a cooldown window.
Once a rule fires and a webhook is successfully sent, subsequent matches within the cooldown window are skipped.
After the cooldown window expires, the rule fires normally again.

**Tradeoff:** Cooldown state is not persisted across service restarts. The cooldown window resets when the service starts.
This is an accepted tradeoff for simplicity; persistent state would require a database or local storage.

**Use case:** Reduce alert spam for noisy conditions (e.g., a contract in a failed state that repeatedly triggers alerts).

```toml
[[contracts.rules]]
type                = "LargeTransfer"
threshold_xlm       = 10000
cooldown_seconds    = 3600  # Alert at most once per hour
```

## Evaluation order

Rules are evaluated in the order they appear in the config file.
All matching rules fire; there is no short-circuit.

## Webhook payload fields

| Field              | Type        | Always present | Description                              |
|--------------------|-------------|----------------|------------------------------------------|
| `label`            | string      | yes            | Contract label from config               |
| `contract_id`      | string      | yes            | Stellar C-address                        |
| `network`          | string      | yes            | `mainnet` / `testnet` / `futurenet`      |
| `rule_triggered`   | string      | yes            | Human-readable rule description          |
| `transaction_hash` | string      | yes            | Stellar transaction hash                 |
| `function_name`    | string/null | no             | Soroban function name if available; `null` indicates a non-Soroban transaction |
| `amount_xlm`       | u64/null    | no             | Transfer amount in XLM if available      |
| `timestamp`        | i64         | yes            | Unix timestamp (seconds) of transaction  |
| `horizon_link`     | string      | yes            | Direct link to transaction on Horizon    |
| `explorer_link`    | string      | yes            | Stellar Expert explorer link for the transaction |

> `horizon_link` and `explorer_link` are always present in every alert payload, even when `function_name` is `null` for a non-Soroban transaction.

## Stable rule_type values

The webhook payload includes two rule-related fields:

| Field | Purpose | Example |
|-------|---------|---------|
| `rule_type` | Machine-readable, stable rule variant name; use for programmatic routing | `"LargeTransfer"` |
| `rule_triggered` | Human-readable description with parameters; use for display | `"LargeTransfer(>=10000XLM)"` |

### Rule type table

| Rule | `rule_type` value |
|------|-------------------|
| `AnyTransaction` | `"AnyTransaction"` |
| `TransactionFailed` | `"TransactionFailed"` |
| `TransactionSucceeded` | `"TransactionSucceeded"` |
| `LargeTransfer` | `"LargeTransfer"` |
| `SmallTransfer` | `"SmallTransfer"` |
| `FunctionCalled` | `"FunctionCalled"` |
| `AdminFunctionCalled` | `"AdminFunctionCalled"` |
| `HighFee` | `"HighFee"` |
| `MemoContains` | `"MemoContains"` |

## Adding a new rule type

1. Add a variant to `AlertRule` in `crates/config/src/lib.rs`
2. Add field validation in `AlertRule::validate()` in the same file
3. Add the match arm in `eval_rule()` in `crates/rules/src/lib.rs`
4. Add the label string in `rule_label()` in the same file
5. Add a stable `rule_type` string in `rule_type()` in the same file
6. Add unit tests in `crates/rules/src/lib.rs`
7. Update the rule type table in this section
8. Update the webhook payload example in README.md (if adding a new example)

No other crates need changes.
