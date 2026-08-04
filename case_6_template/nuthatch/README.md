# Case 6: nuthatch

[nuthatch](https://github.com/nightswatchhq/nuthatch) is a single-binary, self-hosted EVM indexer
written in Rust. It reads the chain over plain JSON-RPC and writes to an embedded store (redb for the
hot tip, Parquet segments past finality, DuckDB for analytical SQL). There is no Postgres, no Docker,
no hosted data service and no API token in this implementation.

## The implementation

There isn't one, in the usual sense: there are no handlers and no code. The whole of case 6 is
declared in `nuthatch.toml`.

```toml
[[contracts]]                 # the factory
alias = "factory"
address = "0x5c69bee701ef814a2b6a3edd4b1652cb9cc5aa6f"
start_block = 19000000
events = ["PairCreated"]

[[templates]]                 # what a discovered child is
name = "pair"
abi = "abis/pair.json"

[[factories]]                 # the rule connecting them
watch = "factory"
event = "PairCreated"
child_param = "pair"
template = "pair"
start = 19000000
```

Children are discovered at runtime and decoded into shared `pair__*` tables keyed by an `address`
column. The discovery is deterministic and is rebuilt from the stored factory events on restart.

Two configuration notes that materially affect the numbers, so they are stated rather than buried:

- **`abis/pair.json` is trimmed to the `Swap` event on purpose.** A `[[templates]]` block takes no
  event allowlist, so its ABI is the allowlist. With the full UniswapV2Pair ABI this nest also decodes
  `Sync`, `Mint`, `Burn`, `Transfer` and `Approval`, producing 72,201 rows: a different workload, not a
  slower one.
- **`block_timestamps = false`.** Case 6 stores no timestamp, and fetching one costs a serial
  block-header round trip per block. Nuthatch only buys that column when a nest declares it wants it.

## Running it

```sh
cargo install --git https://github.com/nightswatchhq/nuthatch nuthatch

export RPC=https://your-mainnet-endpoint/
nuthatch bench backfill --dir . --from 19000000 --to 19010000 --runs 5 --seal-direct --rpc "$RPC"
```

`bench backfill` runs the real fetch/decode/seal path over the pinned range and prints a report
(`--out` writes it to a file) carrying wall clock, event count, peak RSS, RPC request count, provider
host, hardware and the git commit.

To query rather than time it, `nuthatch dev --dir . --seal-direct --rpc "$RPC"` serves an HTTP API and
a SQL endpoint over the same data.

## Results

Median of 5 runs, nuthatch 1.0.1, 11-core laptop (18 GB RAM), Alchemy endpoint:

| | |
|---|---|
| processing time | **49.5 s** |
| records | **35,271** |
| pairs discovered | **232** |
| RPC requests | **16** |
| peak RSS | **247 MB** |

**35,271 = 35,039 `Swap` + 232 `PairCreated`**, so the swap count matches this case's expected 35,039
exactly. Verified independently of the indexer, straight off the RPC: `PairCreated` over the range
returns 232 pairs, and `Swap` filtered to exactly those 232 addresses over the same range returns
35,039.

Artifacts, including the raw report JSON, are in the nuthatch repo under
[`docs/bench/obib-case6.json`](https://github.com/nightswatchhq/nuthatch/blob/main/docs/bench/obib-case6.json).

### A caveat on wall clock

Measured on a shared public endpoint, this range varied between **17 s and 57 s** across a single day
on an identical commit. We tested the obvious explanation rather than assuming it: re-running against
an adjacent, never-fetched range of the same size landed in the same band, so provider caching does not
account for the fast runs. The record count and the 16 RPC requests were invariant across every run.

We would suggest the same caution for any wall-clock figure in this benchmark measured against a shared
endpoint, including ours. Also worth noting for fair comparison: implementations backed by a vendor's
own pre-indexed network are not doing the same work as one reading from RPC, in either direction.

## Hardware

This was run on a laptop rather than the standardised environment, so the number is not
apples-to-apples with the hosted rows. Happy to re-run it wherever the maintainers prefer.
