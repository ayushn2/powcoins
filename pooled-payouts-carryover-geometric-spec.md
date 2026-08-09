# pooled payouts: carryover geometric (work-clock, reward-weighted)

A design for distributing pool income to miners, suitable both for
ordinary mainnet mining via ckpool and for the powcoins stratum server.
The approach is a **carryover geometric** method: each worker holds a
single decaying score denominated in pool work, scores are **never reset
when a block is found**, and each confirmed reward is distributed in
proportion to scores as of the moment the solution was found.

This revises and replaces the wall-clock EWMA spec, which itself revised
the height-tranche PPLNS spec. It keeps the load-bearing properties of
both (custodial, backward-looking, share-weighted, anchored at solve
time, deterministic and publicly replayable, one job only: split
realized income) and changes the accounting clock from wall-time to
pool work, which repairs the EWMA design's residual defects (inexact
exit symmetry for large miners, stall edge cases, stationarity-only
fairness claims) and removes machinery (the snapshot ring and its
retention horizon).

Two parameters: **f**, a flat fee, and **L**, the memory length in
expected blocks of pool work.

## Lineage: Meni Rosenfeld's geometric method, without the reset

The geometric method scores shares with per-share exponential decay and
achieves hopping-proofness: every share's expected reward is a constant,
independent of round age or pool history, because the number of shares
until the next block is geometrically distributed — memoryless — at
every instant. Its awkward parts are consequences of resetting scores
each round: round-length variance in the normalizer must be absorbed by
an operator-held "infinite prehistory" pseudo-share, which makes the fee
variable and bundles a variance-absorption service into the payout rule.

The single change here is **carryover**: one global share sequence,
decay per unit of pool work, no reset at blocks. Consequences:

* **Deterministic denominator.** Past startup, total decayed score is
  identically L (in the units below) — not in expectation, always. The
  pseudo-share's job disappears.
* **Exact fair EV.** A share's expected reward is exactly
  (1−f)·(d/D)·v·B̄ — fair value times its template factor (v ≡ 1
  without job declaration; see Weighting) — with no distributional
  assumptions about round endings: a share's future decay-weighted
  exposure is deterministic and identical for every share; the only
  randomness left is block arrival itself.
* **Flat fee.** The operator takes f of realized income, holds no
  capital, and absorbs no variance. Variance-smoothing is a financing
  layer's product (see the PPHR spec), not the pool's.
* **Per-share smoothing, correctly attributed.** Each share collects
  from ~L future blocks rather than one round (per-share
  CV = 1/√(2L)). For steady members this is invisible — their income
  is L-invariant (see "Choosing L") — so it matters only at the
  boundaries of participation: trials, ramps, exits.
* **The uniform-kernel sibling.** A retained-share PPLNS window over
  the last N of work (e.g. TIDES) is the same design on the same work
  clock with a uniform kernel instead of an exponential one; at
  matched loan (L = N/2) the two have identical per-share variance and
  nearly identical reward distributions, and differ in delivery
  schedule (front-loaded ramp with a tail vs linear ramp with a hard
  finish), share-log retention (a full window of identifiable shares —
  which enables a per-share spot market — vs one number per worker),
  and the absence of window edges (which is what makes every transient
  here a single smooth closed form).
* This is one endpoint of the reset-vs-carryover dial that Meni's
  double geometric method interpolates; the pure-carryover endpoint is
  the one consistent with a distribute-only pool.

## Why the work clock (and not wall-time or share-count)

The score decays per unit of **pool work measured in expected blocks**:
a share of vardiff d at network difficulty D advances the clock by d/D.

* **Decay clock = reward clock.** Blocks arrive, in expectation, at
  rate 1 per unit of this clock regardless of pool size, difficulty
  era, or hashrate fluctuations. This is what makes the EV result exact
  and unconditional where the wall-clock EWMA's equivalent was
  stationary-case-only and size-dependent.
* **Pool hashrate appears nowhere.** A share sent to a 1%-of-network
  pool and to a 10% pool have identical EV; the small pool's memory is
  simply 10x longer in wall-clock, automatically. Inserting a hashrate
  estimate to steer the wall-clock rate would (a) feed a pool-observed
  quantity back into the accounting, violating the no-feedback rule,
  and (b) hand large miners partial control of everyone's decay.
* **Exit symmetry is exact.** Decay and income tick together, so a
  departed worker's outstanding credit ("tail") is redeemed at fair
  value however slow the diminished pool becomes in wall-clock. The
  wall-clock EWMA's large-miner cliff-exit forfeiture vanishes
  identically; the entry-ramp deficit and the exit tail are the same
  quantity (L blocks of income) and cancel.
* **Stalls freeze coherently.** No shares, no decay, no income —
  recovering the original tranche spec's freeze-under-stall property
  that the wall-clock revision had surrendered.
* **Difficulty is the exogenous exchange rate.** Raw stored work is
  converted at read time through a per-retarget difficulty table, so
  stored state is difficulty-agnostic and a retarget mid-memory is
  handled exactly.

The cost, stated honestly: the memory's wall-clock length scales as
L/α at pool fraction α, so a small pool's newcomer loan matures slowly
in days even though it is exact in value. That is a liquidity property,
not a fairness one, and it is what a financing layer prices.

## Parameters

* **L** — memory length, in expected blocks of pool work. Equivalently:
  the steady-state sum of all decayed scores; the payout normalizer;
  the newcomer's loan and the exiter's tail (both exactly L blocks of
  fair income); the mean payout latency; and the direct analogue of
  PPLNS's N (this scheme is "pay the last N of work" with exponential
  rather than uniform weights — but note the loan of a uniform window
  is N/2, so L is comparable to N/2, not N). The per-unit-work
  retention c = e^(−1/L) is derived, never stored as the primary
  parameter. Reference value: L = 3 (see "Choosing L").
* **f** — flat fraction of each claim's value to margin. Load-bearing
  twice: operator revenue, and the tax that makes fee-stuffing
  EV-negative (see Weighting). The payout transaction's fee also comes
  from margin.
* **λ/L epoch log** — L may be changed as operating policy. Changes are
  appended to a small log (workstamp, L), take effect only at the
  current work counter (never retroactively), and are fraction-
  preserving at the instant of change: uniform decay-rate changes leave
  every worker's share of the total untouched, so no payout jumps. The
  only economic effect is a bounded one-shot recalibration of the
  book's outstanding-credit value, of order L·|ln(L_new/L_old)| blocks
  of income, favouring incumbents on increases and future arrivals on
  decreases. One-off uniform decays of all scores are payout-invariant
  (numerator and denominator scale together) and are permitted as
  cosmetic normalization of the published total.
* **Difficulty epoch table** — (workstamp, D) entries appended per
  retarget (pinning L in block units automatically) or on a published
  cadence/threshold (e.g. quarterly, or whenever D has moved more than
  a stated percentage). Either policy is fine; pick one and state it.
  Between syncs L drifts as D moves; the drift never touches fairness
  (see the calibration invariance below), only the advertised memory
  length, and each sync is the same bounded ln-ratio recalibration.
* **Dust threshold** — unchanged from prior specs.
* **The one rule, unchanged:** no parameter may adjust itself based on
  observed pool hashrate or income. Difficulty and the subsidy schedule
  are exogenous and may drive automatic entries; anything else is
  manual, infrequent, and logged.

### The calibration invariance

In steady state, one unit of decayed score represents exactly one
block-of-income of outstanding credit, independent of L: remaining
expected collection per unit weight is ∫ e^(−t/L)/L dt = 1. Scores are
denominated in income owed, not in historical hashes, which is why
mid-flight L changes and difficulty drift are benign: they change how
fast the book turns over, never what it is worth.

### Choosing L

The decisive fact is that **a steady member's income is L-invariant**:
their fraction of the deterministic denominator is pinned at their
hashrate fraction, so they receive fraction × (1−f) × value on every
pool block, and the only variance they observe is block arrival itself
— which no distribute-only kernel can touch. The per-share dispersion
(CV = 1/√(2L)) never surfaces as income variance for anyone whose
shares overlap their own memory window. L therefore buys steady members
nothing, and its magnitude is set entirely by the boundaries of
participation:

* **Floor: trial fairness.** For a share deposited by a brief trial of
  the pool, the probability that the first subsequent block arrives
  only after the share has decayed below fraction q of its weight is
  exactly q^L (first arrival Exp(1) in block units). At L = 1, one
  trial in ten finds its score 90% gone before any payout; at L = 3
  that outcome is one in a thousand, and at least one block lands
  within one memory with probability 95%. Past L ≈ 4–5 the
  improvements are decimal places. The same exponent is the left tail
  of the full per-share reward distribution, which has a closed form:
  reward/fair = (1/L) × generalized-Dickman(L), continuous, no zero
  atom, left tail ∝ x^L. (The uniform-window analogue is
  Poisson(N)/N — a lattice with a zero atom of e^(−N).)
* **Ceilings: granularity, loan, latency.** A small miner's
  entitlement arrives as ~1/L of each share's value per block, so
  large L slices trial rewards into per-block crumbs below the dust
  threshold — paid with near-certainty but experienced as lottery
  draws. The loan and the mean payout latency scale linearly in L
  while buying members nothing.
* **Frontier and kernel choice.** For any distribute-only kernel,
  per-share variance and loan are rigidly linked: CV² = 1/(2·loan)
  (the exponential with mean L and the uniform window of N, loan N/2,
  both sit exactly on this frontier — at matched loan their full
  per-share reward distributions nearly coincide). Kernels therefore
  differ only in delivery schedule. Against a uniform window of N
  blocks, the exponential's newcomer ramp opens at N/L times the
  uniform's payout rate and its worst point is a dip of e^(−N/L) at
  the window edge; e.g. L = 3 against N = 8 opens at 2.7×, stays ahead
  until ~7.3 blocks, dips at most 7%, and does so from a 25% smaller
  loan — near-pointwise dominance, priced at a wider per-share left
  tail that only trial miners can observe.

Hence the reference choice L = 3, with L ≈ 3–5 the principled range:
the floor is the binding constraint, and everything above it pays
loan, latency, and dust-granularity for smoothing no participant can
see. (For comparison against FPPS: FPPS is the zero-loan ramp achieved
off this frontier entirely, by the operator fronting capital and
absorbing variance — i.e. a distribute-only pool with a built-in
financing layer, priced in its fee and in counterparty exposure, and
incompatible with the received-=-paid identity. This design keeps that
product separate; see the PPHR spec.)

## Weighting: work, times an sv2 template factor

Each accepted share deposits weight

```
w = (d / D) · v
```

where d is the share's vardiff (`diff`, the expected work, never
`sdiff`), D the network difficulty at arrival, and v a dimensionless
**template quality factor**:

* **v ≡ 1 wherever the pool selects the template.** Without Stratum v2
  job declaration there is no variation among workers — every share is
  mined on the pool's own template — so the base scheme is pure
  work-weighting. This covers plain ckpool deployments and powcoins
  (where the scheduler picks the job for everyone). It also keeps
  conditional-payoff pricing (hazard/survival) out of the payout
  contract on powcoins, exactly as the custodial rationale requires.
* **v = min(R_own, R_pool) / R_pool under sv2 job declaration**, where
  R_own is the declared template's reward (subsidy + fees, computed by
  the pool at validation, never taken as asserted) and R_pool the
  pool's published per-job benchmark template reward. The full design —
  the cap's policy-veto semantics and automatic subsidy exemption, the
  benchmark ratchet over declared templates, fee-stuffing and
  out-of-band-fee analysis, R_pool publication and audit — is in the
  sv2 job-declaration spec. The governing rule in both cases: *weight
  shares by the expected income the miner's own choices contribute;
  where miners make no choices, that reduces to work.*

Properties needed by the rest of this spec, whichever form v takes:

* **v is dimensionless and ≤ 1** (≡ 1 without JD), so the book sits at
  L·E[v] ≤ L identically through every income regime — fee eras,
  spikes, and the halving move income, not weights — and the published
  total M reads as L times the hashrate-weighted average template
  quality (exactly L without JD).
* **v is exogenous to income.** It is computed from template contents
  against a published contemporaneous benchmark (keyed by job id, not
  wall time), never from realized pool income or hashrate — the
  no-feedback rule applies to v as to every other parameter.

### The halving boundary (and every downward income step)

Because weights are pure work times a dimensionless factor, the book
carries no income denomination at all, and the scheme inherits the
subsidy-normalized boundary profile automatically: post-halving shares
are paid exactly fair immediately (no entry window, no incentive to
prefer a fresh pool), and the boundary's entire cost falls on
pre-halving deposits, whose EV declines smoothly to 50% of deposit-time
fair at the boundary itself, over a window of ~L pool-blocks (≈ L/α
network blocks). (Under sv2 job declaration the same holds because v is
a contemporaneous ratio — R_pool halves with everyone's R_own.) The
alternative (weights carrying raw sats, w ∝ d·R) splits the same
conserved total ~69%/69% across both sides instead; the totals are
identical — the weighting can only choose where the cliff's shadow
falls, never remove it. This design prefers making entrants whole (the
decision miners can still make) and accepts the pre-side incumbent
discount, noting: the discount's per-moment depth and wall-clock
duration trade off with pool size (sharp-but-hours at a dominant pool,
shallow-but-days at a small one); risk-adjusted comparison shows solo
is never the defection target, so the effective discount is measured
against competitors' fees, not against zero; and a financing layer can
hedge a scheduled public event trivially ("halving-window ramp
insurance").

## The lifecycle identity and the startup bounty

The payout denominator is the **actual** decayed sum, not the
steady-state constant L, and every claim distributes its full value
less f. At launch the sum is deficient — L·(1 − e^(−W/L)) at pool work
W — so early shares are paid above fair by the factor

```
g(x) = e^x · ln( 1 / (1 − e^(−x)) ),   x = W/L
```

(≈2.4× one memory-eighth after launch, 1.25× at one memory, converged
by ~5 memories; divergent as ln(1/x) near launch, regularized at share
granularity — the continuum rendering of "the first share takes whole
blocks"). This bounty is monotone (pool work only grows; nothing
regenerates it, so it is not hoppable), fully funded (the block is
distributed either way; received = paid holds from block one), and a
deliberate recruitment feature for young pools.

It is also not free money, by the following identity. A share's EV is
the forward-smoothed income ~L blocks ahead of deposit, so downward
income steps (halvings) under-pay their approach windows and the pool's
final L blocks of tails truncate unredeemed at shutdown. Telescoping
over any income path:

```
L·R_launch  =  Σ_steps L·ΔR_step  +  L·R_shutdown
```

**The startup bounty exactly pre-funds every future halving deficit
plus the shutdown truncation.** The ledger closes per pool lifetime,
not per boundary. All first-moment transients are the one dimensionless
curve family in x = (pool work)/L — startup g(x), pre-halving
1 − e^(−x)/2, shutdown 1 − e^(−x) — so L is a pure scale on the EV
geometry; the only non-scale-free quantity is the per-share CV
1/√(2L) — and per "Choosing L", even that is observable only at the
boundaries of participation, which is why L's principled range is set
by the trial-fairness floor rather than by smoothing.

## State and implementation

* **Per-worker row: (m, t).** m is accumulated weight, t the integer
  workstamp of last update. Lazy decay on touch: decay m across the
  epoch segments in (t, T], add w, set t = T. Idle rows are never
  touched and underflow gracefully toward zero. Reads at any workstamp
  ≥ t are exact in closed form, so any past-index evaluation needs no
  snapshot.
* **Aggregate row (M, t).** Same lazy update, one extra add per share.
  M is the payout denominator, the published audit scalar (should
  hover near L; publish continuously on a short grid, not only per
  payout), and the reconciliation target for a periodic
  recompute-and-compare float-hygiene sweep.
* **Work counter T** — integer, in diff-1 share units (or coarser);
  u128, since large pools exceed u64 within decades. Only the single
  per-touch decay evaluation is floating point, over a small exact ΔW.
* **Epoch tables** — (workstamp, D) and (workstamp, L), append-only,
  effective-forward-only; store cumulative log-decay per entry so any
  read is two lookups and one exponential rather than a segment walk.
* **Per-share cost:** one row update, one aggregate update, one counter
  bump.

## Payouts

1. **Eager capture at solve.** When ckpool submits a solution and it is
   accepted (stale submissions included), immediately record the claim
   txid and compute the full entitlement vector from the live table:
   every worker's fraction m_decayed/M as of the counter value just
   before the solving share. Excluding the solving share (and hence the
   solver's final increment) is the decorrelation offset, now part of
   the anchor definition rather than a parameter. The solver is
   otherwise an ordinary worker: solving confers no bonus, since the
   solve is memoryless and carries no entitlement — only contribution
   does. Store the vector conditional on confirmation.
2. **Settlement.** When the claim reaches settle depth, pay the stored
   vector times (1−f)×value. A claim that never confirms is pruned.
   No snapshot ring, no retention horizon, no evicted-snapshot
   fallback exists in this design; the EWMA spec's ring was an artifact
   of lazy-at-confirmation reads and is entirely replaced by eager
   capture. (On mainnet solves are pool blocks and the capture pass is
   trivially rare; on powcoins the cadence is higher but membership is
   small. If capture cost ever mattered, fractions can be computed in
   the growing frame with no decay at all, since uniform decay cancels
   in every ratio.)
3. **Dust lottery.** Unchanged from the prior specs: entitlements below
   D pool into ⌊T/D⌋ with-replacement draws paying exactly D,
   probability proportional to amount owed, seeded by
   SHA256(confirming block hash at settle depth ‖ claim txid) in
   counter mode — expectation-exact and publicly replayable. (Footnote:
   the seed is safe where confirming-block hashes are not freely
   grindable by arbitrary miners, as on default signet; on a
   permissionless chain the confirming miner holds a dust-sized
   grinding lever, noted and accepted.)
4. **Payouts to stratum usernames as addresses**, custodial single pool
   wallet, exactly as before. Address rotation is payout-invariant by
   linearity (the old address's tail keeps collecting; sum over
   addresses is unchanged) — a small free privacy property; financed
   miners mine to financier-controlled addresses, so rotation there is
   the financier's bookkeeping (see PPHR spec).

## Fairness properties and their assumptions

Stated as claims with preconditions, as the regression suite for future
changes:

* **Exact per-share EV.** E[reward per share] = (1−f)·(d/D)·v·B̄,
  unconditionally on pool size, history, round state (there is none),
  and difficulty era — given (i) block arrival memoryless in work,
  (ii) the solve instant not influenceable by miners (see Known
  limitations for the one residual), and (iii) under sv2 job
  declaration, stationary *relative* template quality over ~L for the
  v factor (vacuous when v ≡ 1).
* **Timing neutrality at every miner size.** The denominator is
  deterministic; no bursting, pausing, or scheduling pattern changes
  EV. (The wall-clock EWMA achieved this only for small miners, with
  concavity-penalized bursting for large ones; here the case split
  disappears.)
* **No hoppable state.** No rounds, no share counts, monotone startup
  bounty, no regenerating quantity anywhere.
* **Exact entry/exit symmetry.** Loan and tail are the same L blocks of
  income and redeem at fair value on the shared work clock.
* **Participation linearity.** EV is linear in hashrate pointed at the
  pool; the 0–100% allocation decision is purely profit-vs-reliability.
  (On powcoins, income inelasticity adds a genuine self-dilution term
  for large entrants — economics, not accounting; disclose it.)
* **Income-regime shifts** (fee eras, halvings, difficulty drift) are
  properties of the reward process, not the accounting; the scheme's
  standard is adding no incentives beyond those, which it meets, with
  the boundary distributions chosen and disclosed above.

## Auditability

* **Self-verification.** A miner reproduces their own numerator exactly
  from their own share stream (weights are d/D from public epoch
  tables; under sv2 job declaration, also their own declared templates
  and the published per-job R_pool series); fraction = own m /
  published M. No sharelog needed.
* **The denominator has two external anchors.** Published M must hover
  near L (structural constant) and cohere with the observed block-find
  rate over time. Inflating M with phantom weight is the only
  profitable operator fraud and is bounded by how much "bad luck" the
  operator will publicly claim, with detection power growing in solve
  count; reshuffling an honest total among sybils is unprofitable
  (uniform self-dilution). Continuous publication (short grid, e.g.
  10 minutes — also the PPHR settlement oracle) makes M a standing
  claim every future payout must stay consistent with.
* **Full determinism retained.** Sharelogs + chain + published tables
  reproduce every payout including lottery draws.
* **Operator-discretion principle.** No mechanism may route funds to
  margin on a classification only the operator can evaluate.
  Miner-side gaming of this scheme only ever redistributes bounded
  amounts among members; operator-side gaming breaks the
  received-=-paid identity that is the design's entire trust story. A
  mitigation that trades the first exposure for the second is rejected
  at any exchange rate. (This is why late solves are not penalized —
  see below.)

## Known limitations, and deliberate non-mitigations

* **Solver solve-withholding.** The anchor is the solve's submission
  point, and the solver alone controls when the pool learns of a
  solve. A miner filtering candidate solves can delay submission while
  their score ramps, capturing a larger fraction. Eager capture already
  kills the *other* timing attack (bursting before a predictable
  confirmation); this one is documented rather than mitigated, on
  three grounds. (a) **Mainnet: EV-negative by staleness.** Delay
  costs ~1−e^(−δ/600) of the entire block to orphaning (~10% per
  minute); the score-ramp gain accrues over pool-blocks of wall-clock,
  i.e. hours. Withholding loses within the first minute at any
  parameters in this design's range. (b) **Enforcement was worse than
  the disease.** The candidate rule — route solves on
  long-superseded jobs to margin — pays the operator on an
  unauditable, operator-only observation (arrival vs. supersession
  times), violating the operator-discretion principle. Rejected.
  (c) **Signet: the vulnerability is the instrument.** powcoins
  solutions never go stale, so (a) inverts and withholding is free
  there — and the deployment is a testbed whose product is these
  fairness claims. The pool therefore *logs* the solve-vs-job-
  supersession lag distribution (observability without enforcement:
  no money moves on it), making in-the-wild withholding measurable;
  the structural fix is an **oblivious-share coin-script family**
  (share/secondary predicate split per the oblivious-header draft,
  deployable on powcoins by minting to new scripts, no fork), whose
  arrival should visibly collapse the lag distribution. The attack
  surface doubles as the experiment.
* **Pre-halving incumbent discount** (above): conserved, disclosed,
  bounded at 50% at the boundary, hedgeable upstairs.
* **Small-pool loan duration**: exact in value, slow in wall-clock;
  a financing-layer product, not a payout defect.

## Database tables

* scores(worker, m, t)
* aggregate(M, t) — one row
* epoch_D(workstamp, D, cumulative_log_decay)
* epoch_L(workstamp, L, cumulative_log_decay)
* rpool_log(job_id, workstamp, R_pool, template_hash) — sv2 job
  declaration only (see the sv2 spec)
* pending_claims(txid, anchor_workstamp, entitlement vector) — pruned
  on invalidation
* payouts(txid, worker, amount, lottery_flag)
* published_totals(workstamp, M) — short-grid series
* solve_lag_log(txid, submit_time, job_supersession_time) — instrument
  only
* param_log(date, name, old, new, note)

Retention: pending_claims rows live only solve-to-settlement; epoch,
payout, published-total, lag, and param rows are small — keep
indefinitely as the audit trail. There is no share buffer and no
snapshot ring.

## Deployment notes

* **ckpool/mainnet**: w = d/D (v ≡ 1); epoch_D written per retarget or
  per stated policy; the payment service remains a pure observer of
  sharelogs and chain (the one-way dependency is unchanged). Adding
  sv2 job declaration activates v per the sv2 spec, with R_pool
  published per job.
* **powcoins/signet**: w = d (v ≡ 1 since the scheduler picks the job;
  D degenerate to a manual constant, epoch-logged); banked-solve
  handling, custodial rationale, and the scheduler's independence from
  share data all carry over from the EWMA spec unchanged; solve-lag
  logging on; oblivious coin-script family as the planned follow-up
  experiment. Phantom-template sv2 job declaration (see the sv2 spec)
  activates v on signet with simulated fees.
* **PPHR compatibility**: the settlement oracle is the published M
  series plus a worker's self-computed score; note the semantic shift —
  with a deterministic total, score measures *fraction of pool work*,
  not absolute hashrate, so financier rate conversion uses the public
  pool work rate.

## Addendum: L in cumulative difficulty, not expected blocks

The spec above defines L in "expected blocks of pool work" and uses a
separate difficulty epoch table (epoch_D) to convert between pool work
and the work counter.  This addendum revises the parameterization:
**L is denominated directly in cumulative difficulty** (the sum of
ckpool vardiff values), absorbing D entirely.

### Revised parameters

* **L** — memory length, in cumulative difficulty.  The e-folding
  constant of the exponential decay, measured in the same units the
  work counter accumulates (sum of vardiff).  On mainnet, set L = k·D
  where k is the desired memory in expected blocks (reference: k = 3,
  so L = 3·D); update L at retargets by appending to epoch_L.  On
  powcoins, set L directly to the desired memory depth in cumulative
  vardiff (e.g. for ~2 h memory at 1 TH/s: L ≈ 1.68e6).  All
  "expected blocks" references in the body above read as L/(D·2³²) on
  mainnet; the fairness, symmetry, and invariance results are
  unchanged since they depend only on L being a constant over the
  memory window.
* **mindiff** — runtime quantization parameter (default 0.1).  The work
  counter stores cumulative difficulty in units of mindiff: each
  accepted share of vardiff d increments the counter by ⌊d/mindiff⌋.
  Chosen per deployment to balance counter granularity against
  compaction frequency (see below).
* **Difficulty epoch table (epoch_D)** — eliminated.  D is absorbed
  into L; the only epoch table is epoch_L.

### Revised decay formula

```
decay(t → T) = exp(-(T − t) · mindiff / L)
             = exp(-(T − t) / (L / mindiff))
```

where T and t are workstamps (work counter values).  Equivalently:
L/mindiff is the e-folding distance in counter units.

### Work counter representation

The work counter T is a **u64 integer**, not u128.  64-bit range is
sustained by periodic compaction:

* **Compaction trigger:** when T reaches 2⁶².
* **Operation:** subtract 2⁶¹ from every workstamp (T, all per-worker
  t values, epoch_L entries, pending_claims anchors).  For per-worker
  scores whose adjusted workstamp would go negative: decay the score
  forward to workstamp 0 and keep the row; only delete if the decayed
  score is exactly zero.  For epoch_L entries whose adjusted workstamp
  would go negative: delete all except the most recent (highest
  workstamp), which is clamped to workstamp 0 with its
  cumulative_log_decay advanced to the adjustment point:
  `new_cld = old_cld + (adj − old_ws) · mindiff / old_L`.
  The current L must always be retained.
* **Correctness:** subtraction preserves all (T − t) differences for
  normally-shifted entries, so their decay calculations are exactly
  unchanged.  The clamped entry's CLD update preserves decay
  correctness across the collapsed segments (CLD differences, which
  are what decay_between computes, are unchanged).
* **Deletion safety:** deleted rows have (T − t) ≥ 2⁶¹ in pre-shift
  counter units.

### The L/mindiff constraint

Require **L/mindiff ≤ 2⁵⁷**.  This guarantees that any data deleted by
compaction has decayed through at least 2⁶¹/2⁵⁷ = 16 e-folding times
(residual weight < 1e-7).  Enforced at startup and at every L change.

### Revised database tables

* epoch_D — deleted (D absorbed into L)
* epoch_L(workstamp, L, cumulative_log_decay) — unchanged
* All other tables unchanged; "workstamp" columns store the u64 work
  counter value

### Revised deployment notes

* **ckpool/mainnet:** L = 3·D (updated at retargets via epoch_L);
  mindiff chosen so rescale headroom (2⁶¹·mindiff·2³²/pool_hashrate)
  is comfortable (mindiff = 0.1 gives ~46 days at 250 EH/s).
* **powcoins/signet:** L set directly (no D); mindiff = 0.1 gives
  effectively infinite headroom at TH/s-scale pools.

### Dust lottery: win amount

Each draw pays `pool_total // n_draws` (where `n_draws = pool_total // D`),
not exactly D.  This can exceed D when `pool_total` is not an exact
multiple of D, but it conserves sats within the dust pool rather than
silently routing the remainder to margin.

### Lottery seed: containing block hash

The mechanism section says "SHA256(confirming block hash at settle
depth ‖ claim txid)".  The block hash used is the **containing block**
(the block in which the claim transaction was mined), not the block
whose arrival pushes confirmations past the settle-depth threshold.
The containing block hash is permanently tied to the transaction and
deterministic from the chain alone.

### Batched dust lottery seed

When multiple coins settle in the same run, the implementation
concatenates all `(blockhash || txid)` pairs and hashes once, rather
than running a separate lottery per coin.  This produces a different
seed than the per-claim `SHA256(blockhash || txid)` described in the
mechanism section.  The batched variant is deterministic and replayable
given the full set of settling coins; the per-claim formula applies
only when a single coin settles alone.

### Capture cutoff

The eager-capture step (Payouts §1) skips workers whose decayed score
is below 1e-10.  This is a storage optimization: at any realistic L,
1e-10 represents a vanishingly small fraction of the denominator and
cannot produce a nonzero payout in sats.  The cutoff keeps the
candidate_payouts table compact without affecting any payout amount.
