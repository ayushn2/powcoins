# delta: pack-based deadlines (replaces "Deadlines (deferral)" and parts of "Main loop")

The per-job deferral test (`maturity > C * 2**d / h_est`) is only right
when grind windows don't overlap. With several jobs wanting roughly the
same stretch of hashrate, deadlines must be checked against the time to
finish *each prefix* of the work, not each job alone. So: schedule as a
whole, one slot per coin, and compute slack for the schedule.

## Terminology

* **tier** -- a specific difficulty `d` for a coin. Higher d = harder
  (more hashes) but matures sooner (fewer halvings to wait). Lower d =
  easier (fewer hashes, higher logMV) but matures later.
* **anchor** -- the coin's best-logMV tier: the committed fallback that
  determines the slot's deadline (= the anchor's maturity).
* **upgrade** -- a harder tier (higher d) of a coin that already has a
  slot. Attempted opportunistically while slack permits. If it doesn't
  solve in time, the slot degrades back toward the anchor.

## Grind budget

```python
def G(mus):
    """Blocks needed for ~95% chance of solving ALL the given jobs,
    grinding them back-to-back. mus = [2**d / h_est for each job].

    Each job's solve time is exponential (memoryless hashing) with
    mean mu. The sum has mean sum(mus) and standard deviation
    sqrt(sum(mu**2)), and we take the 95th percentile as mean + 2*sd.

    Why 2: for a single exponential the exact 95th percentile is
    -ln(0.05) ~= 3*mu, and since sd = mu for an exponential,
    mean + 2*sd = 3*mu exactly. For many equal jobs the CLT kicks in
    and 95% needs only mean + 1.64*sd, so 2 is a few percent
    conservative. Errs early, never late, by <= ~4%."""
    return sum(mus) + 2.0 * sum(m * m for m in mus) ** 0.5
```

## Schedule feasibility

A schedule is a list of slots sorted by deadline (earliest first). Each
slot represents one coin with an anchor tier and a current grind target
(which may be harder than the anchor if the slot has been upgraded).

The first k slots must all be solved by the k-th slot's deadline, so
every prefix imposes a constraint:

```python
def slack(schedule, now, h_est):
    """Returns how many blocks of room the schedule has.
    > 0: room to admit more work
    == 0: fully committed; start now
    < 0: over capacity (see escape valve)"""
    mus = []
    s = INF
    for slot in schedule:
        mus.append(2.0 ** slot.d / h_est)
        s = min(s, slot.deadline - G(mus) - now)
    return s
```

## Building the schedule

All (coin, d) pairs are sorted by logMV globally. The build loop walks
this list best-first. The first hit for a given coin creates a new slot
(**anchor**); subsequent hits for the same coin are **upgrades** -- they
replace the slot's grind target with a harder tier while keeping the
anchor's deadline, since the anchor remains the fallback.

```python
def build(coins, now, h_est):
    schedule = []
    for c, d in candidates_by_logMV(coins):      # best first
        trial = schedule_with(schedule, c, d)
        # schedule_with: if c is new, insert a slot at d's deadline;
        # if c already has a slot, upgrade its grind target to d
        # (keeping the anchor's deadline).
        if slack(trial, now, h_est) >= 0:
            schedule = trial
        # else: doesn't fit -- but keep going; a later candidate with
        # a smaller grind or a friendlier deadline may still fit.
    return schedule
```

This spends slack on **breadth** (new coins) and **depth** (harder tiers
on existing coins) in plain logMV order, until slack runs out or
candidates do.

Example: coin C has anchor d=47 (best logMV). Later in the candidate
list, d=48 for coin C appears (lower logMV, but harder and matures
sooner). If slack permits, the slot upgrades: it now grinds d=48, but
the deadline stays at d=47's maturity, because d=47 is the fallback if
d=48 doesn't solve in time.

## Time-based degradation

An upgraded slot doesn't stay at its hardest tier forever. As time
passes and the upgrade tier's own maturity passes without a solve, the
slot falls back one tier at a time toward the anchor: 48 -> 47. At each
step the grind target gets easier (half the hashes), making it more
likely to complete before the anchor's deadline.

## What to grind right now

```python
def current_work(schedule):
    if not schedule:
        return None         # no live coins: report no-work to pool
    return schedule[0]      # earliest deadline, at its current grind target
```

If the schedule has slack after build(), that just means we ran out of
worthwhile candidates before running out of time. Idle hashrate has no
value, so grind the earliest-deadline slot anyway.

Work never stops because a time budget "ran out": past hashing doesn't
change the next hash's expected value. Work changes only when
displaced -- a rebuild reorders the schedule, a solve lands, a coin dies.

## Escape valve (slack < 0)

If any slot has been upgraded past its anchor, revert it one tier (e.g.
d=48 -> d=47): halves the slot's mu, freeing slack at the cost of giving
up the harder tier's upside. Revert the lowest-logMV upgraded slot
first.

If no upgrades remain and slack is still negative, drop the
lowest-logMV slot entirely.

If still negative, accept the overrun -- a deadline isn't a wall, the
coin just sits at its current tier for extra blocks, costing hazard the
table can price.

## Events (unchanged list)

Rebuild schedule + recompute slack on: new block, our solve (replace
that coin's slot with d_best+1 insurance or drop it), competitor claim,
coin appears, h_est moves materially. A solve frees grind window, slack
reopens, next-best candidates get admitted.

## Notes

* The old per-job `C * 2**d / h` budget is just `G([mu])` -- the k=1
  special case -- so nothing else in the scheduler doc changes
  numerically.
* 95% is a heuristic, not a guarantee: per-prefix 95% isn't joint 95%,
  but missing prefix k is highly correlated with missing k+1, so joint
  coverage sits just under the binding prefix's. Early solves donate
  their leftover window to later slots via rebuilds, so realized
  coverage runs higher than the formula suggests.
* G is deliberately a touch conservative for mixed packs. If exactness
  ever matters: Wilson-Hilferty with effective shape
  (sum mu)**2 / sum(mu**2) gets sub-2% everywhere; not worth it now.
