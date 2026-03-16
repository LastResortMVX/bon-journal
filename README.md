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

## Managing wallets across shards

- **Loading:** All PEMs in `--wallets-dir` are loaded and grouped into three lists: shard 0, shard 1, shard 2. Every command that uses wallets sees them already split by shard.
- **Intra-shard:** For `transfer-intrashard` we only send between wallets in the **same** shard (receivers chosen at random within that shard). Relayers, when used, must also be in that shard.
- **Cross-shard:** For `transfer-cross-shard` we send from wallets in a **source** shard to wallets in a **destination** shard. Relayers are taken from the **source** shard.
- **All shards at once:** If you omit `--shard` in `transfer-intrashard`, the script runs **one thread per shard**. Shard 0, 1, and 2 are processed in parallel; within each shard, every wallet sends one batch of `--tx-per-wallet` transactions. That way we drive load on all three shards simultaneously from a single run.

So “managing several wallets across shards” here means: one wallet dir, automatic shard assignment, and commands that either target one shard, or run all shards in parallel with one thread per shard.

---

## Multicore logic

We use Python’s `ThreadPoolExecutor` in two ways:

1. **One thread per shard (when `--shard` is omitted in `transfer-intrashard`, or in `test-batches`)**  
   Each shard that has wallets gets one worker. So with wallets in shards 0, 1, 2 we get three threads; each thread iterates over its shard’s wallets and sends one batch per wallet. No cross-shard locking: each thread only touches its own shard’s wallets and nonces.

2. **N threads within a single shard (`--threads N` in `transfer-intrashard` when `--shard` is set)**  
   We split the wallets of that shard into N chunks and assign each chunk to a worker. Each worker builds and sends batches for its wallets (one batch of `--tx-per-wallet` tx per wallet). So on a multi-core machine you can use e.g. `--threads 4` or `--threads 8` to send from many wallets in parallel inside one shard.

Nonces are either fetched once from the network at the start or loaded from a file (`--nonces-file`). After that, each sender’s nonce is incremented in memory as we build transactions; we do not re-query the network between rounds when using `--loop`.

---

## Loop and batch behaviour (how it was used)

- **Fund:** One transfer per wallet from the whale. Tx are sent in batches of 100; we wait for the last tx of each batch to complete before sending the next batch.
- **Intra-shard (one shard):** One “round” = each wallet sends one batch of `--tx-per-wallet` tx (default 100). With `--loop`, we repeat rounds with a delay `--round-delay-ms` (default 600 ms, applied after the round time). Multicore: `--threads` splits wallets across workers.
- **Intra-shard (all shards):** Same idea, but one thread per shard; each shard runs its own “every wallet sends one batch” round. With `--loop`, all shards sleep together then run the next round.
- **Cross-shard:** One shot: each source-shard wallet sends 99 tx to random destination-shard wallets. All tx are built then sent in batches of 1000 (no wait between batches).
- **test-batches:** One batch per wallet across all shards; shards run in parallel (one thread per shard). Used to sanity-check that we can send one batch from every wallet on all shards.
- **test-swap:** One wrap, then swap tx in batches of 100, waiting for the last tx of each batch before the next. Single wallet.

---

## Setup

```sh
python3 -m venv ./venv
source ./venv/bin/activate
pip install -r ./requirements.txt --upgrade
```

---

## Commands and parameters (reference)

### `create-wallets`

Create N wallets once; no loop, no shards.

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--wallets-dir` | yes | — | Directory to save PEM files (created if missing) |
| `--number-of-wallets` | yes | — | Number of wallets to create |

---

### `fund`

One pass: one EGLD transfer per loaded wallet from the whale. Batches of 100; wait for last tx of batch before next batch.

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--wallets-dir` | yes | — | Directory containing wallet PEM files |
| `--whale` | yes | — | Path to whale PEM |
| `--network` | yes | — | Network URL |
| `--amount` | no | all balance | Total to distribute (atomic units) |
| `--gas-price` | no | network | Gas price in atomic units |

---

### `transfer-intrashard`

Intra-shard transfers; multicore via `--threads` (single shard) or one thread per shard (omit `--shard`). Optional `--loop` with `--round-delay-ms`.

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--wallets-dir` | yes | — | Directory containing wallet PEM files |
| `--network` | yes | — | Network URL |
| `--shard` | no | all | Shard (0, 1, or 2). Omit to run all shards in parallel |
| `--amount` | yes | — | Amount per transaction (atomic units) |
| `--relayer` | no | — | Relayer address (same shard). Mutually exclusive with `--random-relayer` |
| `--random-relayer` | no | false | Use a random relayer from same shard per tx |
| `--loop` | no | false | Run rounds until Ctrl+C |
| `--threads` | no | 1 | Worker threads (single-shard mode) |
| `--round-delay-ms` | no | 600 | Delay (ms) between rounds when `--loop` |
| `--tx-per-wallet` | no | 100 | Transactions per wallet per round |
| `--nonces-file` | no | — | Load nonces from JSON/CSV (from `fetch-nonces`) |
| `--gas-price` | no | network | Gas price in atomic units |

---

### `transfer-cross-shard`

One shot: 99 tx per source wallet to random destination-shard wallets. Batches of 1000, no wait between batches.

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--wallets-dir` | yes | — | Directory containing wallet PEM files |
| `--network` | yes | — | Network URL |
| `--source-shard` | yes | — | 0, 1, or 2 |
| `--destination-shard` | yes | — | 0, 1, or 2 (must differ from source) |
| `--amount` | yes | — | Amount per transaction (atomic units) |
| `--relayer` | no | — | Relayer from source shard. Mutually exclusive with `--random-relayer` |
| `--random-relayer` | no | false | Random relayer from source shard per tx |
| `--gas-price` | no | network | Gas price in atomic units |

---

### `fetch-nonces`

Fetch current nonce for every loaded wallet; write to JSON or CSV. No loop.

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--wallets-dir` | yes | — | Directory containing wallet PEM files |
| `--network` | yes | — | Network URL |
| `--output` | yes | — | Output path (`.json` or `.csv`) |

---

### `test-batches`

One batch per wallet on all shards; shards in parallel (one thread per shard).

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--wallets-dir` | yes | — | Directory containing wallet PEM files |
| `--network` | yes | — | Network URL |
| `--amount` | yes | — | Amount per transaction (atomic units) |
| `--tx-per-wallet` | no | 100 | Transactions per wallet in the single batch |
| `--nonces-file` | no | — | Load nonces from JSON/CSV instead of network |
| `--gas-price` | no | network | Gas price in atomic units |

---

### `test-swap`

One wrap, then swap tx in batches of 100; wait for last tx of each batch. Single wallet.

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--wallet` | yes | — | Path to wallet PEM |
| `--amount` | yes | — | WEGLD per swap (atomic units) |
| `--number-of-swaps` | no | 1 | Number of WEGLD→USDC swaps |
| `--network` | yes | — | Network URL |
| `--min-amount-out` | no | 1 | Minimum USDC per swap (slippage) |
| `--relayer` | no | — | Relayer (same shard). Mutually exclusive with `--random-relayer` |
| `--random-relayer` | no | false | Random relayer per tx |

---

## Summary: batch and round behaviour

| Command | Batch size (send) | Waits between batches? | Rounds / loop |
|---------|--------------------|-------------------------|---------------|
| `fund` | 100 | Yes (last tx of batch) | 1 |
| `transfer-intrashard` | 1 batch per wallet (`tx-per-wallet` tx) | No | 1 or `--loop` |
| `transfer-cross-shard` | 1000 | No | 1 (99 tx per wallet) |
| `test-batches` | 1 batch per wallet (`tx-per-wallet` tx) | No | 1, shards in parallel |
| `test-swap` | 100 (swaps only) | Yes (last tx of batch) | 1 wrap + 1 swap phase |

Amounts are in atomic units (1 EGLD = 10^18). Relayers must be in the sender’s shard (source shard for cross-shard).
