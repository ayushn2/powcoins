# powcoins pool: work selection design

What work should the pool grind, given N faucet coins with different
values, difficulties and halving rates?

## Definitions

* `d` -- log2 difficulty, 16..80. A solution at difficulty `d` needs
  `2**d` hashes in expectation.
* coin -- a faucet utxo, with:
  * `v = lg(value_in_sats)`
  * `r` -- blocks per difficulty halving (1, 4 or 10)
  * `d_now` -- current difficulty
  * `t` -- blocks until next halving (`1 <= t <= r`)
  * `d_best` -- difficulty of the best (lowest-d) solution we already
    hold for this coin, or `None`
* job -- a pair `(coin, d)` with `d <= d_now`: a signature whose
  nSequence allows spending once the coin decays to difficulty `d`.
  A solution found for a job can't be broadcast until the coin actually
  reaches `d`; it stays valid forever after that (until the coin is
  spent), so solved jobs are kept indefinitely.
* `h` -- our hashrate, in hashes per block interval.
* `H[dv_bin, r]` -- hazard table: per-block probability that a coin
  sitting at difficulty `d` (so `dv = d - v`) is spent by someone else.

Key fact used throughout: hashing is memoryless. A hash is an
independent lottery ticket; there is no progress, no benefit to
finishing early, and abandoned work loses nothing already "built up".

## Hazard table

Tracks how quickly other people spend coins, as a function of how
attractive the coin is (`dv = d - v`; cheap-per-value coins get sniped,
so hazard is roughly a function of dv) and halving family `r`.

Two counters per bin, updated once per signet block:

```python
def on_block(coins, claims):
    for c in live_coins:
        E[bin(c.d_now - c.v), c.r] += 1          # exposure: coin-blocks
    for c in claims_by_others:
        K[bin(c.d_now - c.v), c.r] += 1          # events: snipes
    # our own claims just remove the coin; no K increment

def hazard(dv, r):
    return (K[bin(dv), r] + K0) / (E[bin(dv), r] + E0)   # prior K0/E0 > 0
```

Notes:
* width-1 dv bins are plenty; optionally split each sample across the
  two straddled bins in proportion (v is non-integer).
* the prior `K0/E0` is the "assume some competition before we've seen
  any" knob.
* optionally apply a decay factor to E and K every block so old
  behaviour fades (competitors come and go).
* log raw `(d, v, r)` events too, so the binning can be redone later.

## Survival

`q(coin, d)` -- probability the coin is still unspent when it decays to
difficulty `d`, walking the known schedule from now:

```python
def q(c, d):
    lam = c.t * hazard(c.d_now - c.v, c.r)          # rest of current tier
    for dd in range(d + 1, c.d_now):                # full tiers in between
        lam += c.r * hazard(dd - c.v, c.r)
    return exp(-lam)
# q(c, c.d_now) == 1; q decreases as d decreases.
```

## Job value: logMV

Expected sats per hash for grinding job `(c, d)`, in log2:

```python
def logMV(c, d):
    if c.d_best is None:
        dq = q(c, d)
    else:
        if d <= c.d_best:
            return -inf      # held solution fires first; dominated
        # probability the coin survives to d but NOT to d_best,
        # i.e. the only worlds where this new solution matters.
        # computed as q(c,d) * (1 - exp(-lam_between)) to avoid
        # subtracting two numbers both near 1:
        lam = sum(c.r * hazard(dd - c.v, c.r) for dd in range(c.d_best + 1, d)) \
              + partial_tier_terms(c, d, c.d_best)
        dq = q(c, d) * -expm1(-lam)
    return c.v - d + lg(dq)
```

Properties worth remembering:

* fresh coin: each step deeper (d -> d-1) doubles the cost, so it pays
  iff crossing that tier loses less than half the survival probability,
  i.e. iff `r * hazard(d - v, r) < ln 2`. The best d sits one notch
  *into* wherever the hazard spikes, not at the safe edge.
* after a win at `d_best`: everything deeper than `d_best` is dominated
  (cancel it); the only same-coin work left is `d_best + 1` insurance,
  whose dq is the hazard of the single crossing. It competes in the same
  queue as other coins' jobs and is only attractive where the table is
  actually hot.

## Hashrate estimate

Only used for deadlines, so +/-50% is fine. Estimated from solves vs
exposure of issued work (no per-share data needed):

```python
# on issuing/refreshing work: accumulate exposure
exposure += blocks_outstanding * 2.0**-job.d
# on each solve:
h_est = ewma(solves_seen / exposure)
```

Seed `h_est` deliberately low: too-low h opens deadlines early (cheap),
too-high opens them late (overrun blocks spent at hot tiers). Early
solves correct it quickly. A few minutes pointed at some stale coin's
shallow tier produces fast solves and is a cheap recalibration whenever
the estimate looks stale.

## Deadlines (deferral)

Grinding a job early buys nothing (memorylessness + can't broadcast
before maturity) and burns the option to redirect those hashes if the
coin dies or something better appears. So each job has a start-by time
and is **deferred** until then:

```python
def maturity(c, d):                  # blocks until job (c,d) is broadcastable
    return c.t + c.r * (c.d_now - 1 - d)

def grind_blocks(d):                 # time budget to solve with confidence
    return C * 2.0**d / h_est        # C ~ 2..3

def deferred(c, d):
    return maturity(c, d) > grind_blocks(d)
```

Jobs at or past maturity (including `d == d_now`) are never deferred.

## Main loop

```python
def best_job():
    jobs = [(c, d) for c in live_coins
                   for d in range(lowest_sane_d(c), c.d_now + 1)
                   if not dominated(c, d) and not deferred(c, d)]
    return max(jobs, key=lambda cd: logMV(*cd), default=None)
    # near-ties: prefer earlier maturity (it's the perishable one)

# recompute best_job and reissue work on any of:
#   - new signet block (difficulties decay, t ticks, hazard table updates)
#   - a coin appears or is spent
#   - one of our jobs solves (update d_best; cancel dominated work)
#   - h_est moves materially (~25%+)
```

Emergent behaviour to expect (useful as a sanity check on an
implementation): for a fresh coin the deepest non-deferred tier wins,
so the pool steps down through tiers as their deadlines open
(`d_now - d ~ C * 2**d / h`); transit through each tier yields ~1+
expected solves, so first wins land early and most of a coin's life in
the queue is then a `(d_best, d_best+1 insurance)` pair drifting down
the schedule. With a single coin and nothing else to grind, deferral is
moot -- idle hashrate has no opportunity cost, so just grind the best
job regardless.

## ckpool-facing notes

* virtual height (prevhash preimage) should bump -- forcing a work
  restart and stale-flagging -- only when outstanding work is actually
  worthless: our own solve on the current job, or the coin being mined
  got spent. Other queue changes (different coin became best, new coin,
  hazard reshuffle) should reuse the height so in-flight work isn't
  flagged stale.
* keep a persistent, append-only map: job id -> (coin outpoint, d,
  sig, tx skeleton). Stale solves are still valid claims while the coin
  is unspent, and this map is the only job->coin audit trail.
* ckpool submits stale solves; worth an explicit test that a
  deliberately-stale solve still lands as a claim after any ckpool
  upgrade.
* worker-level divergence (stale-by-one-refresh jobs) is harmless:
  scores near the top are flat and stale jobs are still fair tickets.
  Only grinding *dominated* work costs anything, bounded by one refresh
  interval per dominance event.

## Deliberately ignored (for now)

* payouts (parked; note the claim-tx-carries-the-split variant would
  put share-awareness into the powcoins server).
* confirmation-race margin (delta blocks at maturity) -- set to 0.
* depletion discount for work already committed -- refresh cadence
  makes it negligible.
* capacity packing when several deadlines collide -- if it shows up,
  sort colliding jobs by maturity and check prefix sums of grind
  budgets; shading one job to d+1 halves its budget for ~15% MV and is
  often the cheaper fix.
