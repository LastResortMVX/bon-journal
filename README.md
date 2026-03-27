# BON Journal — Challenge 4

---

## What we were doing

We extended the workload beyond plain transfers and DEX rings:

- **`forwarder-cross-shard` + `--use-forwarder-relayers`** — This is where the **three distinct relayer wallets (one per shard)** live in `start.py`: constants **`RELAYER_0`**, **`RELAYER_1`**, **`RELAYER_2`**. PEMs for those addresses must sit under **`--wallets-dir`**; the loader maps bech32 → `Account` and fills **`ForwarderConfig.relayers_by_shard`**. In **`worker_thread`**, each tx uses **`shard_relayer = relayers_by_shard.get(node.shard)`**: senders on shard 0/1/2 get the relayer for that shard only. **`kwargs["relayer"]`** and **`sign_as_relayer(tx, shard_relayer, …)`** use that account — not the ring wallets, not **`--whale-pem`**.
- **`forwarder-cross-shard`** (contract calls) — Each ring node calls its **shard’s forwarder contract** (`ForwarderConfig.contracts_by_shard[node.shard]`). Endpoints mix **blind** functions per shard; swap schedule is **50/50 WEGLD↔USDC** per batch plan (shuffled). Relayers are optional; without **`--use-forwarder-relayers`**, those relayer lines are skipped.
- **`relayed-cross-shard` / `relayed-dex`** — **Different relayer model in code**: a **single** account loaded from **`--whale-pem`** (`whale_account` in the source). **`relayer_addr`** is always **`whale_account.address`** for every tx; **`sign_as_relayer(tx, whale_account, …)`** runs for all relayed sends. There is **no** shard-indexed relayer map in these modes — only the forwarder path uses the three per-shard keys above.

The common engine is unchanged in spirit: **load PEMs → shard via `AddressComputer` → build a ring → partition across `--threads` → batches of up to 100 signed txs → `send_transactions`**, with nonce locks, partial-send rollback, and the suspension / nonce watcher when failure rates spike.

---

## How the script wires relayed vs forwarder logic

**Ring topology (`build_ring`)**

- **`forwarder-cross-shard`** uses the **same global ring** as `normal` / `dex` / `relayed-dex`: wallet *i* → wallet *i*+1. The “destination” in the node is still the next wallet; **forwarder mode ignores that for the SC path** and instead targets **`ForwarderConfig.contracts_by_shard[node.shard]`**.
- **`relayed-cross-shard`** keeps the **interleaved cross-shard ring** (like `cross-shard`). Relayer v2 uses **one** **`whale_account`** from **`--whale-pem`** for every transaction (no per-shard switch).

**Relayed v2 (`--whale-pem` — single relayer for all shards)**

For `relayed-cross-shard` and `relayed-dex`, after building each transaction, **`sign_as_relayer(tx, whale_account, transaction_computer)`** runs. Native and DEX builds pass **`relayer=whale_account.address`**.

**Forwarder path (shard contracts + three shard relayers when enabled)**

`ForwarderConfig` holds, per shard: **forwarder contract address**, **owner** (drain / ops), **`relayers_by_shard`** (populated from **`RELAYER_0` / `RELAYER_1` / `RELAYER_2`** when **`use_relayers`**), plus gas limits and swap parameters.

Per transaction in **`worker_thread`** (forwarder branch):

1. Pick **`endpoint`** from a per-shard **function plan** (e.g. shard 1 uses 100× `blindSync`; other shards mix `blindAsyncV1`, `blindAsyncV2`, `blindTransfExec`) and **`token_in` / `token_out` / `amount_in`** from the **100-slot swap plan** (50 WEGLD→USDC, 50 USDC→WEGLD), both **shuffled** per batch for variety.
2. Build **`function_call_parts`** for the inner DEX call (`swapTokensFixedInput` + token out + min out as raw byte buffers).
3. **`create_transaction_for_execute`** on the **shard forwarder** with arguments **`[exchange_contract, function_call_parts]`** and **`token_transfers`** of the chosen token/amount; **`gas_limit`** is the large forwarder budget (`FORWARDER_GAS_LIMIT`).
4. If **`use_relayers`**, **`shard_relayer = relayers_by_shard.get(node.shard)`** — attach **`relayer=shard_relayer.address`** and **`sign_as_relayer(tx, shard_relayer, …)`**. Forwarder mode is **not** in **`RELAYED_MODES`**, so **`whale_account`** is never used here.

**Drain / redistribute (optional in code)**

There is a **`forwarder_drain_thread`** / **`_drain_and_redistribute_shard`** design (owner calls **`drain`** on the forwarder, then redistributes ESDT to shard wallets). In the current tree it may be commented out at startup; operationally it matters for keeping forwarder liquidity balanced across long runs.

---

## Gas price: what failed us today

The script still uses a **single compile-time `GAS_PRICE`** for built transactions (native, wrap/unwrap, SC executes). It does **not** periodically refresh **`min_gas_price`** from **`get_network_config`**, and it does **not** bump price on systematic “gas too low” or gateway rejections beyond the existing nonce rollback / suspension behaviour.

**What failed us today was not the ring or relayer wiring — it was the lack of planning around gas price evolution.** As the network floor or competition moved, a fixed price turned into silent friction: partial accepts, rejections, and wasted batches while we treated the symptom (nonces, gateway load) instead of **tracking and adapting to a moving minimum**.

For a future run, the operational minimum would be:

- Refresh **network min gas price** (or a safe multiple) on an interval or before large batches, and feed that into **`gas_price`** for new transactions.
- On **persistent** low-acceptance or explicit low-gas errors, **raise** price and **retry** after rollback, without assuming yesterday’s constant still clears the mempool today.
- Revisit **cost estimates** (`_gas_price_per_tx`, **`--whale-pem`** balance for relayed modes, **`RELAYER_0/1/2`** balances for forwarder+relayers) whenever `GAS_PRICE` or limits change — relayed and forwarder paths multiply small price deltas by **very large gas limits**.

---

## Quick command map (this repo)

| Intent | Mode / flags |
|--------|----------------|
| Cross-shard ring, **one** relayer (all shards) pays gas | `--mode relayed-cross-shard --whale-pem <single-relayer.pem>` |
| DEX ring, **one** relayer pays gas | `--mode relayed-dex --whale-pem <single-relayer.pem>` |
| Forwarder + **three** relayers (shard 0/1/2) | `--mode forwarder-cross-shard --abi ./forwarder-blind-bon.abi.json --use-forwarder-relayers` — PEMs for **`RELAYER_0`**, **`RELAYER_1`**, **`RELAYER_2`** in **`--wallets-dir`** |

Forwarder contract addresses, owner addresses, and **`RELAYER_*`** bech32 values are **constants in `start.py`** (`RELAYER_0` … `RELAYER_2`); those three PEMs must be present in **`--wallets-dir`** when **`--use-forwarder-relayers`** is set.

---
