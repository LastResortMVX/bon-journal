# BON Journal

Notes on how these scripts were used to manage many wallets across shards, with multicore logic for intra-shard and cross-shard traffic.

---

## What we were doing

We had a set of PEM wallets spread across shards 0, 1, and 2. The goal was to:

- **Fund** all of them from a single whale.
- **Generate load** per shard (intra-shard transfers) and **between** shards (cross-shard transfers), with optional **relayers**.
- Use **multiple CPU cores** so we could send from many wallets in parallel without blocking on a single thread.

The scripts load every `.pem` from a directory, assign each wallet to a shard via the address (MultiversX SDK `AddressComputer`), then run commands that respect shard boundaries and optional relayer/sender logic.

---

## How the script works (implementation)

The driver we used in this repo is **`start.py`**: a continuous MultiversX transaction generator built on the SDK’s `ProxyNetworkProvider` and `TransfersController`. Conceptually it matches the behaviours described in the command reference below (funding, intra-shard, cross-shard, DEX, relayers); the concrete CLI uses **`--mode`** instead of separate subcommands.

**Pipeline**

1. **Load wallets** — Every `*.pem` under `--wallets-dir` becomes an `Account`; `AddressComputer` assigns each address to shard 0, 1, or 2.
2. **Connect** — One gateway base URL is passed as **`--network`** (see [Two machines and gateways](#two-machines-and-gateways)). The script reads chain ID and, unless `--empty-nonces` is set, **fetches each account’s nonce** from the proxy (in parallel).
3. **Build a ring** — Wallets are wired into a **ring topology** that depends on mode:
   - **`normal` / `dex` / `relayed-dex`** — One global ring: wallet *i* sends to wallet *i*+1 (mod *n*).
   - **`intra-shard`** — One independent sub-ring **per shard** (only same-shard hops).
   - **`cross-shard` / `relayed-cross-shard`** — Wallets are **interleaved** round-robin across shards so each hop crosses to another shard where possible.
   - **`wrap` / `unwrap`** — Each wallet targets its shard’s wrap contract.
4. **Shard the work across cores** — Ring **nodes** are **partitioned round-robin** across **`--threads`** worker threads. Each worker only cycles its own subset of nodes (so multiple threads can touch different senders concurrently).
5. **Send loop** — Each worker repeatedly walks its nodes. For each node it builds **up to 100 signed transactions** in one go (one batch), then calls **`send_transactions`** on the provider. There is no per-batch wait for finality in this path: throughput is limited by signing, HTTP, and gateway acceptance. Optional **`--limit`** stops after that many transactions have been reported sent; otherwise the run continues until Ctrl+C.
6. **Relayed modes** — For `relayed-cross-shard` and `relayed-dex`, a **`--whale-pem`** wallet signs as the **relayer** (gas paid by the whale per relayed v2 rules).

So “the script” here is: **load → nonce sync → ring → N workers → batches of 100 tx** until stop. The tables in [Commands and parameters](#commands-and-parameters-reference) stay the human-readable map of *what* we wanted to do; **`start.py`** is the ring-based engine we actually ran for sustained load.

---

## Two machines and gateways

We ran the workload from **two different computers** to split CPU and outbound connections. Each machine had its own checkout, venv, and a **disjoint subset** of the wallet PEMs — we **split wallets across machines** so each address was only ever driven by one host. The script has **no built-in coordination** between hosts; sharing the same PEMs on two boxes would risk **nonce clashes** (two in-memory sequences for one on-chain account). Splitting the set avoids that.

**Kepler vs public gateway**

The script takes a **single** `--network` URL. We used the **Kepler gateway** as the primary endpoint for lower latency and competition-specific routing. When Kepler was **slow, erroring, or rate-limiting**, we **restarted** the process with the **public MultiversX gateway** URL as `--network`. There is **no automatic failover** in code: recovery is **operational** (restart or shell wrapper that probes Kepler and switches URL). Both endpoints speak the same proxy API; only the base URL changes.

---

## Nonces, gateway rejects, and automatic recovery

**Why nonces go out of sync**

Each sender’s next nonce must match what the **chain** expects. The script keeps a **single in-memory nonce per `Account`**, advanced when building transactions. Parallel workers use **one lock per sender address** so two threads never reserve the same nonce for the same wallet. If the **gateway accepts only part** of a batch, or **rejects** transactions (wrong nonce, low gas, temporary errors), the in-memory counter can drift ahead of reality until corrected.

**What happens on send**

- Transactions are built with **`get_nonce_then_increment()`** so nonces are reserved sequentially inside the lock.
- **`send_transactions`** returns how many transactions were actually accepted (`num_sent`) and may return fewer than the batch size if the gateway drops some.
- **Partial acceptance:** For each **missing** acceptance, the code **decrements** the account nonce by that count so the next batch retries the **skipped** nonces.
- **Total failure** (exception path): the whole batch is rolled back by decrementing by the batch length.
- **Signing/build errors** for a single tx: decrement by one for that attempt.

So “refused by the gateway” shows up as **partial or zero `num_sent`**; the client **rolls back** the unused nonces immediately.

**Automatic recovery when things go badly wrong**

If **at least ~75%** of the last batch **failed** to send (gateway saturation, persistent bad nonce, etc.), the wallet is **suspended** for that worker: the ring **skips** it until recovery. A background **`_nonce_watcher`** thread polls the account on the proxy about every **2 seconds**. When the **on-chain nonce** has **caught up** to what we consider the baseline **and** balance is still enough for gas, the wallet is **resumed**. If balance is too low, the watcher logs a **low balance** state and the wallet can stay suspended. This avoids hammering the gateway with impossible nonces while the chain or another client advances the account.

**Fresh start from the network**

On startup, **`initialize_nonces`** pulls current nonces from the proxy (unless **`--empty-nonces`**, which forces 0 for special cases). There is no continuous nonce sync during steady state except the suspension path above.

Together, this gives **per-batch rollback**, **per-wallet suspension** under heavy failure, and **reconciliation** via polling before resuming.

---

## Dynamic gas price (not needed for this challenge)

The network exposes **dynamic minimum gas price** (and related parameters) via the proxy’s network config. For **native transfers** in `start.py`, gas price is set to a **fixed** value (`1_100_000_000` in atomic units in the current code path), not re-read on every batch.

During the Battle of Nodes run, **fixed gas was enough**: congestion did not force us to chase a moving floor or outbid other senders for inclusion. If we had needed it, sensible optimisations would have been:

- Periodically refresh **`min_gas_price`** (or a small multiple) from **`get_network_config`** and use that for new transactions.
- On **systematic gateway rejection** or **“gas price too low”** style errors, **bump** gas price and **retry** the same nonces (after rollback).
- Optionally track **recent inclusion** or proxy hints, if available, to avoid overpaying while staying above the floor.
- For **DEX / SC** paths, align with whatever the `TransfersController` / SC factory uses by default and apply the same refresh strategy.

None of that was implemented for this challenge because the fixed setting remained viable end-to-end.

---

