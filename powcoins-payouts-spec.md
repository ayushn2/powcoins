# powcoins pooled payouts

A design for paying miners who point hashrate at the powcoins stratum
server. The approach is PPLNS (pay-per-last-N-shares) with shares
grouped into tranches, exponentially decaying tranche weights, and
payouts anchored to the moment a solution was found.

* All claim transactions pay a single pool-controlled signet wallet.
* A separate **pool payment service** reads ckpool's sharelogs and
  watches the chain, and distributes confirmed rewards to miners
  according to their recent share history.
* The payment service only observes; it sends no data to the powcoins
  server or to ckpool, so the existing one-way dependency between those
  components is unchanged. Miners set their stratum username to a
  signet address, which is where their payouts go.
* Every payout is a deterministic function of the sharelogs and the
  final chain, so anyone holding both can verify the pool's accounting,
  including the dust lottery draws.

## Background: why this shape

* **Custodial.** A solution is a conditional payoff tied to one
  specific coin (it may never confirm if someone else spends the coin
  first). Any non-custodial scheme has to put a value on those
  conditional payoffs, which drags the scheduler's hazard and survival
  estimates into the payout contract. Paying everything to a pool
  wallet and distributing the income that actually arrives avoids
  pricing anything.
* **Anchored at solve time.** Because the spend signature commits to an
  nSequence, a solution can be found long before it is broadcastable,
  and the eventual confirmation time is publicly predictable. If
  payouts were based on shares near the *confirmation*, someone could
  point a lot of hashrate at the pool just before a known fat claim
  lands and capture most of it. Solving, by contrast, is memoryless:
  nobody can predict or position for the moment a solution appears. So
  each claim pays the miners who were contributing shares at and before
  its solve.
* **Tranches measured in chain height.** Pool income is capped by the
  faucet's per-block drip, so it arrives on the chain's clock, and
  fairness means comparing one miner's work against everyone else's
  over the same span of that clock. Denominating tranches in blocks
  also means share weights stop decaying whenever signet stalls (when
  no income is arriving either), and every pool instance, large or
  small, gets the same tranche cadence with nothing to configure or
  estimate.

## Parameters

* Tranche span: 6 signet blocks (about an hour at default timing).
* Recognized tranches: 192. Decay: 0.98 per tranche, so the newest
  tranche's weight is 0.02 and weights halve roughly every 34 tranches.
  The weights sum to about 97.94%; the remaining ~2% is pool margin.
* Settle depth: 12 blocks. Tranche boundaries are finalized, and
  payouts triggered, only once buried 12 blocks (covers reorgs up to
  10 deep).
* Dust threshold D: chosen at deployment.
* These values are deployment choices, and any fixed choice is fine.
  The one rule: never let a parameter adjust itself based on observed
  hashrate or income — that feedback is the only way participants could
  influence their own decay. Manual, infrequent changes are fine; log
  them.

## Mechanism

1. **Tranche boundaries.** Tranche k covers block heights 6k up to but
   not including 6(k+1). Its boundary timestamp T_k is the running
   maximum of block header nTime values through height 6k (the running
   max keeps boundaries monotonic, since raw nTime need not be). Write
   each boundary once its height is 12 blocks deep, and never revise
   it.

2. **Shares.** Read shares from ckpool's sharelogs. Credit every share
   ckpool accepted, including stale ones, weighted by its `diff` field.
   Use `diff` (the vardiff target, i.e. the expected work the share
   represents) rather than `sdiff` (the achieved hash quality, which is
   just the same work plus luck — summing it adds variance for no
   benefit). Hold incoming shares in a buffer; once a tranche's closing
   boundary is settled, assign buffered shares to it by createdate
   falling within [T_k, T_{k+1}), record per-worker totals for the
   tranche, and discard the raw shares. The buffer only ever holds
   about two hours of data.

3. **Anchors.** Whenever ckpool submits a solution to the powcoins
   server and it is accepted (stale submissions included — those are
   exactly the early-ground solutions anchoring exists for), record an
   anchor: the claim txid, the solving share's hash, its workinfoid,
   and its createdate. This must be recorded at solve time; it cannot
   be reconstructed later. Which tranche the anchor belongs to is
   worked out when that tranche settles. Within the anchor's own
   tranche, only shares up to and including the solving share count
   toward that claim (shares are ordered by createdate, with workinfoid
   and line number as tiebreaks), so when rolling up the anchor's
   tranche, also store each worker's share total up to each anchor's
   cutoff.

4. **Payout runs.** When a claim transaction reaches 12 confirmations
   and its anchor tranche has been rolled up, distribute its value.
   Step backwards through tranches starting from the anchor tranche
   (step j = 0) to step j = 191, giving step j a weight of
   0.02 × 0.98^j. Each worker receives the claim value times the sum
   over steps of (step weight) × (their share of that tranche's total),
   using the cutoff totals for step 0 and the full tranche totals
   otherwise. If a step's total is zero — an empty tranche, a solve
   that was the first share of its tranche, history from before the
   pool started, or a claim confirming so late its whole window has
   expired — that step's weight goes to margin. The payout
   transaction's fee also comes from margin.

5. **Dust lottery.** Workers whose payout meets the dust threshold D
   are paid directly. Pool the remaining small entitlements (total T)
   and run ⌊T/D⌋ draws *with replacement*, each draw picking a worker
   with probability proportional to what they were owed, each win
   paying exactly D. With-replacement keeps every worker's expected
   payout exactly proportional to what they were owed; whatever is left
   under one D goes to margin. Seed the draws with
   SHA256(confirming block hash at the 12-confirmation point ‖ claim
   txid), extended counter-mode for as many draws as needed, so anyone
   can replay the lottery from public data.

## Database tables

* boundaries(k, T_k)
* share_buffer(createdate, workinfoid, line, worker, diff, hash) —
  transient, dropped at rollup
* tranches(k, worker, diff_total)
* anchors(txid, share_hash, workinfoid, createdate, tranche, status)
* anchor_partials(txid, worker, diff_total) — per-worker totals up to
  the cutoff
* payouts(txid, worker, amount, lottery_flag)
* param_log(date, name, old, new, note)

Retention: tranche and partial rows can be dropped once 192 tranches
older than the newest tranche any unconfirmed anchor could still point
to — roughly 12 days, given a maximum solve-to-confirmation lag of
about 640 blocks (worth re-deriving from the scheduler's actual maximum
grind-ahead). Boundaries, anchors, and payouts are small; keep them
indefinitely as the audit trail.
