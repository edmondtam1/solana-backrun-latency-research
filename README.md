# Solana atomic back-run arbitrage: latency research

A measured look at where the time goes in a Solana back-run arbitrage, from a trigger swap landing to our
transaction landing behind it. The work covers our own Rust engine, data feeds, send routes and the leader's block,
all measured on mainnet with real transactions (November 2026).

**Scope:** atomic, back-run-only arbitrage across Solana DEX pools. Every trade is a single transaction that either
returns more SOL than it spends or fails. There is no front-running, sandwiching, bait launches or wash trading.

## The system

- **Engine:** Rust. It decodes swaps from a Yellowstone gRPC stream and keeps pool state from account
  subscriptions. It quotes 2-pool and 3-leg (hub) cycles across DEX venues including Raydium CPMM and CLMM, Orca
  Whirlpool, Meteora DLMM and DAMM v2, and PumpSwap. It handles Token-2022 transfer-fee mints.
- **Location:** a single box in Frankfurt (AWS eu-central-1). Unprotected sends go only to leaders within about 10 ms;
  far-leader slots aren't contestable from one region.
- **Safety:** a signer guard refuses any transaction whose minimum output doesn't cover input + tip + fees + a
  profit floor. It also enforces a failure-fee budget, a wallet floor halt, a per-trade size cap and a hot-path
  latency gate before every release.

## Engine latency (receive → signed)

Measured live on mainnet, per opportunity:

| Stage | p50 | p90 | p99 |
|---|---|---|---|
| Decode | 0.15 ms | 0.43 ms | 1.0 ms |
| Detect | 0.95 ms | 3.6 ms | 7.3 ms |
| Signed | 0.66 ms | 2.3 ms | 5.9 ms |
| HTTP hand-off to send endpoint | +1.5 ms | | |

Our own code accounts for about 1–3 ms of the race.

## Feeds

These are delivery delays behind the earliest raw shred, from three feeds measured side by side in Frankfurt over
800k transactions:

| Feed | p50 | p95 | p99 |
|---|---|---|---|
| Decoded shreds | (first in 97.5% of cases) | | |
| Accelerated Yellowstone gRPC | 1.3 ms | 3.8 ms | 6.2 ms |
| Standard managed Yellowstone gRPC | 46 ms | 17 s | 29 s (stalls) |

## Send routes

Each race sent small probe transactions in interleaved rounds, one route per round, with landing position read
from the block (about 27 per route):

| Route | Send → seen on chain (p50) | Position through block (p50) |
|---|---|---|
| Helius Sender (SWQoS tier) | 37 ms | 0.57 |
| Helius Sender (all-pathway tier) | 38 ms | 0.60 |
| Astralane | 42 ms | 0.51 |
| Nozomi / direct QUIC to leader TPU (earlier race) | within noise of Sender | |

The service hop costs about nothing. What remains is the leader's own intake, scheduling and entry batching, and
that's the same for every route.

## Who wins the race

We looked at lost races on-chain, comparing the first competing arb with our transaction.

| | Standard gRPC | Accelerated gRPC |
|---|---|---|
| Our lag behind the winner (median) | 134 txs | 62 txs |
| Losses within 50 txs of the winner | 7% | 39% |
| Winner lands at trigger +1 | 46% | 45% |

- **A faster feed halves our lag**, but it doesn't touch the ~45% of races won directly behind the trigger.
- **Those winners saw the swap before it executed.** That means private order flow, not public speed.
- **Priority fees aren't the gap.** We paid more priority per CU than the median winner and still landed behind.
  Most winners paid no tip at all.

## Us vs the winners

| | Us: standard gRPC | Us: accelerated gRPC | Winners |
|---|---|---|---|
| Sample | 175 races, 2 h | 39 races, 14 min | 120 races (separate day) |
| Landing time after the trigger (median) | ~111 ms | ~57 ms | ~5 ms (all); ~18 ms (excl. trigger +1) |
| Landed in the trigger's block | 52% | 68% | ~80% |
| Races won | 0 / 175 | 1 / 39 | n/a |
| Priority per CU (median) | — | 231k µL | 90k µL |
| Paid a send-service tip | — | — | 27% |
| Net per won race | — | 0.00036 SOL (n = 1) | median ~0.0001 SOL; p75 ~0.0008 SOL |

- **The quotes aren't the problem.** 104 of 122 checked opportunities matched the on-chain result exactly, and the
  one win realised exactly the predicted net.
- **The timing model was pre-registered.** We predicted the accelerated feed's effect before the run. The landing
  time (57 ms vs predicted 35–125) and same-block share (68% vs 62–81%) came in inside the prediction; the win rate
  (2.6% vs 5–33%) fell below it. That points to private flow, not speed.
- **Time conversion:** times come from block positions at ~1.3 transactions per ms, so treat them as estimates.

## Conclusion

- **The public path tops out short of the winners.** On the public path (fastest feed, any send route, competitive
  priority), a well-engineered back-runner from one region still lands behind the trades that matter.
- **The remaining edge sits behind pending-transaction access.** A back-run-only searcher needs a sanctioned channel
  where user flow is protected and users are paid for it.

---

Contact: Edmond Tam ([@edmondtam1](https://github.com/edmondtam1))
