# powcoins pooled payouts (EWMA revision)

A design for paying miners who point hashrate at the powcoins stratum
server. The approach is a continuously-decaying per-worker **score** — an
exponentially-weighted moving average (EWMA) of each worker's
difficulty-weighted share rate — with each claim paying every worker in
proportion to their share of the total score as of the moment the
solution was found.

This revises the earlier height-tranche PPLNS spec. It keeps that
design's load-bearing properties (backward-looking, share-weighted,
hopping-resistant, custodial, anchored at solve time, deterministic and
publicly replayable) and changes the accounting from a fixed-length
chain-height window of summed tranches to a single decaying score per
worker. The motivation is fairer treatment of a miner who ramps hashrate
up or down — see "Why this shape" — and a simpler, smaller, reorg-immune
implementation.

* All claim transactions pay a single pool-controlled wallet.
* A separate **pool payment service** reads ckpool's sharelogs and
  watches the chain, and distributes confirmed rewards to miners
  according to their recent share history.
* The payment service only observes; it sends no data to the powcoins
  server or to ckpool, so the existing one-way dependency between those
  components is unchanged. Miners set their stratum username to a payout
  address, which is where their payouts go.
* Every payout is a deterministic function of the sharelogs and the
  final chain. In addition, each payout publishes a single scalar — the
  total pool score at the solve anchor — so any miner can verify their
  own fraction from their own share stream plus that one number, without
  the full sharelog (see "Auditability").

## Background: why this shape

* **Custodial.** Unchanged from the original. A solution is a
  conditional payoff tied to one specific coin (it may never confirm if
  someone else spends the coin first). Any non-custodial scheme has to
  put a value on those conditional payoffs, which drags the scheduler's
  hazard and survival estimates into the payout contract. Paying
  everything to a pool wallet and distributing the income that actually
  arrives avoids pricing anything.

* **One job: split realized income.** The pool's sole responsibility is
  to divide the income it actually receives fairly across the hashrate
  that earned it. It deliberately does **not** reduce variance in those
  payouts. A miner who wants smoothed income buys that from a financing
  layer on top (e.g. an ecash mint or a PPS reseller that takes the raw
  lumpy stream and sells a smooth one), which is the party that holds
  capital and prices risk. Keeping distribution and financing separate is
  what preserves the clean, no-capital, publicly-auditable accounting
  identity "what was received = what was paid"; the moment the pool
  fronts or smooths money, that identity breaks. Variance therefore
  passes through to miners undamped, by design.

* **Anchored at solve time.** Unchanged in rationale. Because the spend
  signature commits to an nSequence, a solution can be found long before
  it is broadcastable, and the eventual confirmation time is publicly
  predictable. If payouts were based on shares near the *confirmation*,
  someone could point a lot of hashrate at the pool just before a known
  fat claim lands and capture most of it. Solving, by contrast, is
  memoryless: nobody can predict or position for the moment a solution
  appears. So each claim pays the miners who were contributing shares at
  and before its solve. With a live decaying score this anchoring is also
  what makes the scheme un-timeable: the score is read at an
  unpredictable instant (the solve), so a worker cannot burst right
  before the read.

* **Score decays in wall-clock time, not chain height or pool work.**
  The score is an EWMA of difficulty-weighted shares with a fixed
  wall-clock time constant. The decisive reason over a work- or
  height-denominated window is small-pool behaviour. A work-denominated
  window (e.g. "last 8 blocks of work") has a wall-clock memory that
  stretches inversely with the pool's share of total hashrate: on a pool
  that is 0.5% of the network, eight blocks of pool work span ~11 days,
  so a newcomer ramps to fair pay over ~11 days. A time-denominated EWMA
  has the same few-hour memory at any pool size, so the same newcomer
  ramps over hours. For a small pool trying to attract a large new miner
  — the case that matters here — the time-denominated scheme is
  dramatically friendlier. The cost is that the original's
  "weights-freeze-when-the-chain-stalls" property is given up; this is
  acceptable because uniform decay cancels in the payout ratio, and the
  only residual is membership turnover across a long stall (see
  "Behaviour under stalls").

* **Why not pure last-N-shares.** Share-count denomination restores
  entry/exit symmetry (the deficit a ramping-up miner extends equals the
  credit they recover on the way down) but reintroduces dependence on
  pool hashrate in the window length and carries a capital "loan" sized
  in N. The EWMA is the continuous-decay generalization that keeps the
  symmetry (subject to the stationary-block-rate caveat in "Behaviour
  notes"), fixes the memory in wall-clock time, and collapses the whole
  history into one number per worker. The decay constant is the only free
  parameter; choosing it is choosing where to sit on the
  variance-vs-responsiveness curve, nothing more.

## Parameters

* **Decay half-life:** 2 hours. This is the canonical parameter; the
  per-second decay λ = (1/2)^(1/7200) ≈ 0.99990379 is derived from it at
  startup and need not be stored separately (the lazy-decay step is just
  `score *= 0.5 ** (Δseconds / t½_seconds)`). Quote the half-life to
  humans: a worker's score halves every 2 hours of absence, and a worker
  who steps their hashrate reaches `1 − 2^(−hours/2)` of their new steady
  score — 50% after 2 h, 75% after 4 h, 87.5% after 6 h, 93.75% after
  8 h, ~99.98% after 24 h. The "loan" a ramping worker extends to the
  pool is correspondingly about one half-life of their earnings. (For
  reference, t½ = 2 h is τ = t½/ln 2 ≈ 2.89 h; halving the half-life
  doubles responsiveness and halves the loan.)
* **Snapshot interval:** ~10 minutes of wall-clock time. The live score
  vector is frozen on this grid into a ring buffer (see "Snapshots").
* **Snapshot retention horizon:** ~5–7 days. Must exceed the scheduler's
  true worst-case solve-to-confirmation wall-clock lag with margin; see
  "Retention" for sizing and the failure mode.
* **Decorrelation offset:** small, e.g. the snapshot strictly preceding
  the solve time. Ensures the solving share (and its correlated recent
  burst) is not in the score it pays against.
* **Dust threshold D:** chosen at deployment.
* **Margin:** the published total score sums all workers' scores; the
  pool's cut is taken as a flat fraction off each claim's value before
  distribution (the analogue of a fixed fee). There is no variable /
  operator-variance-absorption fee — that machinery solves a financing
  problem the pool does not have.
* These values are deployment choices, and any fixed choice is fine. The
  one rule, unchanged: never let a parameter adjust itself based on
  observed hashrate or income — that feedback is the only way
  participants could influence their own decay. The decay constant in
  particular must be a constant, not a function of pool hashrate. Manual,
  infrequent changes are fine; log them.

## Mechanism

1. **Score.** Maintain one running score per worker. Conceptually each
   worker's score is the EWMA, in wall-clock time, of their accepted
   shares weighted by `diff`. Credit every share ckpool accepted,
   including stale ones, weighted by its `diff` field (the vardiff
   target, i.e. the expected work the share represents) — not `sdiff`
   (achieved hash quality, which is the same work plus luck and only adds
   variance). In implementation the decay need not be applied per second
   to every worker: store each worker's score together with the timestamp
   it was last updated, and when a new share arrives (or at snapshot
   time) decay lazily by 0.5^(Δseconds / t½_seconds) before adding the
   share's `diff`. The score is a lossless summary of the entire weighted
   history; no share log need be retained once its `diff` has been folded
   in.

2. **Snapshots.** On a ~10-minute wall-clock grid, freeze the full score
   vector (all workers' current scores, decayed to the grid instant) into
   a ring buffer, tagged with the snapshot timestamp. Snapshots reference
   neither the chain nor any banked solve — they are pure periodic
   freezes of the score, computed only from the share stream. This
   decoupling is what makes them reorg-immune: a reorg can change which
   claim confirmed and when, but cannot move or invalidate a snapshot,
   because shares are off-chain pool data. The grid being predictable is
   not a gaming risk: the payout trigger (the solve) is unpredictable, so
   an attacker cannot know which snapshot a future solve will pay
   against.

3. **Banked solves.** Whenever ckpool submits a solution to the powcoins
   server and it is accepted (stale submissions included — those are
   exactly the early-ground solutions anchoring exists for), record a
   bare row: the claim txid and the solve timestamp. This must be
   recorded at solve time; it cannot be reconstructed later. Nothing else
   is computed now — no snapshot association, no per-worker rollup. A
   banked solve that never confirms (someone else spends the coin first)
   is simply never processed further. Work scales with confirmations, not
   with the 400–800 solves that may be outstanding at any time.

4. **Payout runs.** When a claim transaction reaches its settle depth in
   confirmations, join it — lazily, now — to the snapshot immediately
   preceding its solve timestamp (the decorrelation offset). Distribute
   the claim's value (less flat margin) to every worker in proportion to
   their score in that snapshot divided by the total score in that
   snapshot. The solver is treated as an ordinary worker: solving confers
   no bonus, so a worker who was not contributing in the reference
   snapshot receives nothing for having found the block — which is
   correct, since the solve is memoryless and carries no entitlement,
   only contribution does. If the snapshot's total score is zero, or the
   required snapshot has already been evicted from the ring (see
   "Retention"), the claim's value goes to margin and the event is
   logged. The payout transaction's fee also comes from margin.

5. **Dust lottery.** Unchanged from the original. Workers whose payout
   meets the dust threshold D are paid directly. Pool the remaining small
   entitlements (total T) and run ⌊T/D⌋ draws *with replacement*, each
   draw picking a worker with probability proportional to what they were
   owed, each win paying exactly D. With-replacement keeps every worker's
   expected payout exactly proportional to what they were owed; whatever
   is left under one D goes to margin. Seed the draws with
   SHA256(confirming block hash at the settle-depth point ‖ claim txid),
   extended counter-mode for as many draws as needed, so anyone can
   replay the lottery from public data.

## Auditability

The score is a low-dimensional, self-computable statistic, which gives a
practical audit story stronger than an unpublished per-share log.

* **Self-verification of your own fraction.** A miner sees their own
  submitted shares completely (it is their own hardware). They reproduce
  their own score exactly with a one-line filter (0.9999/sec over their
  own `diff`-weighted shares). The only quantity they cannot compute
  alone is the denominator (total pool score at the anchor), which the
  pool publishes as a single number per payout. Your fraction =
  your score ÷ published total. No sharelog required.

* **External sanity check on the total.** Total pool score is an estimate
  of total pool hashrate, and pool hashrate has an independent ground
  truth: the block-find rate. The only profitable manipulation —
  inflating the denominator with phantom shares — inflates the implied
  hashrate, which then disagrees with the observed block-find rate over
  time, so it is externally detectable. An honest total reshuffled among
  fake internal workers is undetectable from the score alone but is also
  unprofitable, because each worker's fraction is their score over the
  total and a fake worker inside an honest total dilutes everyone
  (operator included) identically. The only profitable fraud is the
  checkable one; the only uncheckable manipulation is unprofitable.

* **Full determinism retained.** As before, anyone holding the sharelogs
  and the final chain can recompute every payout, including the lottery
  draws. The published per-payout total score is the additional artifact
  that makes the common-case check (does my rolling hashrate match my
  payout) doable from public data plus one's own equipment.

## Behaviour notes

* **Newcomer ramp (worked example).** A miner with 2% of network
  hashrate joins a pool that previously had 0.5% of network hashrate (so
  the miner is 80% of the pool's current hashrate). Their warm-up factor
  after elapsed time is `1 − 2^(−hours/2)`, while the incumbents sit at
  full steady state. If they find a block after only 50 minutes
  (warm-up ≈ 0.25), their payout works out to ~50% of the block rather
  than the 80% that current hashrate alone would suggest; found at a more
  typical time it is far closer to fair. Because the incumbent base here
  is small, the payout *fraction* recovers toward 80% faster than the
  raw warm-up factor: ~66.7% at 2 h, 75% at 4 h, ~77.8% at 6 h, ~79.5%
  at 10 h. A miner who is instead a *tiny* fraction of the pool barely
  moves the denominator, so their fraction tracks the warm-up factor
  directly: 50% of fair at 2 h, 75% at 4 h, 87.5% at 6 h, 93.75% at 8 h,
  ~99.98% at 24 h — effectively fully converged within a day. For
  comparison, an 8-blocks-of-work window on the same 0.5% pool would pay
  the lucky-early-block miner ~1.25% and take ~11 days to converge — the
  small-pool pathology the wall-clock EWMA avoids.

* **Exit is not perfectly symmetric for a large miner.** The entry
  deficit a ramping miner extends is recovered on exit *only if the pool
  keeps finding blocks at its pre-exit rate while the departing score
  decays* — the exit credit is a claim on future blocks, and future
  blocks are found by whoever is still hashing. For a miner small
  relative to the pool this holds (their leaving barely changes the
  pool's block rate), so entry and exit cancel. For a miner large
  relative to the pool it fails: their own withdrawal removes most of the
  block-finding capacity that would redeem the credit, so a cliff-exit
  forfeits the still-elevated tail of their score. Example: the 80% miner
  cliff-exits; the pool reverts to ~0.5% of network (~one block per
  11 days), while their score halves every 2 h, so the credit is almost
  certainly unredeemed before it decays away. Two things bound and
  mitigate this. The short half-life caps the forfeited amount at roughly
  one half-life of earnings (a couple of hours' worth), not days — this
  is the main reason it is tolerable. And a miner who cares can ramp
  hashrate out gradually over several half-lives: at each step they are a
  smaller fraction of a pool still partly powered by their remaining
  hashrate, so each decrement's credit is redeemed against blocks the
  pool is still finding. Honest summary: a small, bounded, large-miner-
  only edge in the miner's disfavour, mitigated by the short half-life
  and by gradual ramp-down.

* **Behaviour under stalls.** Wall-clock decay continues during a chain
  or pool stall. Because decay is uniform it cancels in the payout ratio,
  so ongoing scores are unaffected in relative terms. The only edge is a
  long stall combined with membership turnover across it, and a claim
  whose solve snapshot ages out of the time-denominated ring before the
  claim confirms; pad the retention horizon to cover this. For mainnet
  this is negligible; for signet it is possible but low-stakes.

* **Small-worker floor.** The score is a usable hashrate estimate for any
  worker landing enough of their own shares per τ. With a low vardiff
  floor, even small ASICs find a share every few seconds, so τ ≈ 2.8 h
  contains thousands of a small worker's own shares — comfortably tight.
  Participants too small to clear the floor at a few-second cadence are
  in the noise regime where extra variance does not matter. There is no
  need to provision for solve density: income lumpiness from few solves
  per τ is a financing-layer concern, not the pool's.

## Database tables

* scores(worker, score, last_update) — the live running scores.
* snapshots(snap_time, worker, score) — the ring buffer of frozen score
  vectors; or equivalently a per-snapshot blob keyed by snap_time.
  Bounded by the retention horizon.
* banked_solves(txid, solve_time, status) — bare rows recorded at solve
  time; most never confirm.
* payouts(txid, worker, amount, lottery_flag)
* published_totals(txid, snap_time, total_score) — the per-payout audit
  scalar.
* param_log(date, name, old, new, note)

Retention: snapshots older than the retention horizon are evicted from
the ring. The horizon must exceed the scheduler's true worst-case
solve-to-confirmation wall-clock lag with margin. A practical top-end
solve-to-confirmation lag is ~500 blocks (the 640-block figure assumes
solving at 2^80 hashes per solve, which is uneconomic); at ~10-minute
blocks that is ~3.5 days, so a 5–7 day ring leaves comfortable headroom.
This is a soft parameter with a graceful, observable failure mode: if the
ring is too short, a confirming claim's snapshot lookup misses, that
claim's value is routed to margin, and the miss is logged — a recurring
log entry is the signal to lengthen the ring. Sizing: at ~10-minute
snapshots over 5 days (~720 vectors) and up to ~30k members at ~16 bytes
per worker, the ring is a few hundred MB; far less at smaller membership.
Banked-solve, payout, published-total, and param-log rows are small; keep
them indefinitely as the audit trail.
