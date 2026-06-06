# Pretty Good Pizza Voting

A voting mechanism for repeated games with discrete outcomes, designed for simplicity, fairness, resistance to gaming, and avoidance of two-party (dichotomous) dynamics. The optimization target is average satisfaction across the voter population, not majority preference.

The "pizza" framing is a small concrete example. The intended targets are larger: distributing grants and contracts among broader sets of participants, and selecting representatives or officials without entrenching simple-majority domination.

## Design goals

- **Simplicity.** Voters express preference as approval (yes/no per option). No ranking, no scoring.
- **Fairness.** Outcomes track the actual distribution of approval across the population, not just the winning side.
- **Strategy-resistance.** Honest voting is close to optimal. Coordinated gaming yields amplification proportional to coalition size — no more.
- **Anti-dichotomy.** Broadly-acceptable candidates and minority preferences both retain real weight, instead of being squeezed out by a winner-take-all dynamic.

## Core mechanism

Every variant shares the same front end (approval + threshold) and differs only in the back end (allocation rule). The shared vocabulary:

- `N_voters` voters, `N_options` options.
- Each option has `slices(option)` (units delivered) and `price(option)` (units consumed).
- `value(option) = slices(option) / price(option)`.
- `M` is the total budget — money, seats, awards, whatever the resource is.
- `threshold` is the minimum approval level an option must clear to remain in the running. There is no universal best rule; top-K, fixed fraction, and fixed integer are all reasonable choices.

### Step 1 — Approval voting

Each voter casts `approval(voter, option) ∈ {0, 1}`. Any number of approvals is allowed, including none or all.

### Step 2 — Threshold filter

```
votes(option) = Σ_voter approval(voter, option)
```

Options whose `votes(option)` falls below `threshold` are zeroed out (eliminated).

### Step 3 — Allocation

Choose one of the allocation rules below, depending on the problem.

## Allocation rules

### A. Stochastic, with replacement

Use when the same option can legitimately be selected more than once — a pizza kind can be ordered twice; a vendor can win multiple awards from the same pool.

```
while M ≥ min(price(option)):
    candidates ← { option : price(option) ≤ M }
    pick one option from candidates with
        P(option) ∝ votes(option) × value(option)
    M ← M − price(option)
```

Probability is proportional to `votes × slices / price` — popular *and* efficient options are favored, but neither sweeps.

### B. Stochastic, without replacement

Use for indivisible selection where each option can win at most once — a sole-source contract, a single-seat election, a grant a participant can only receive once.

Same as (A), but each selected option is removed from the candidate pool before the next draw. Terminates when `M` is exhausted or no remaining option satisfies `price ≤ M`.

### C. Deterministic, weighted top-M

Use for representative bodies with `M > 1` seats, where randomness is politically unacceptable.

1. Take the top `M` options by `votes(option)` after the threshold filter.
2. Seat each. Their voting weight inside the body is proportional to their `votes(option)`.

This is equivalent in expectation, over many internal votes, to drawing `M` seats with `P ∝ votes`. It pays the proportionality cost in per-person power concentration instead of in randomness.

Practical notes:
- Procedural rules (majority-present, committee shares, quorum-to-conduct-business) become weight-based, not head-count-based. This requires deliberate redesign.
- An optional weight cap (e.g., no individual exceeds `2/M` of total weight, excess redistributed by approval rank) protects against extreme concentration at some cost to proportionality.
- A vote transform (`sqrt(votes)`, `log(votes)`) is another knob for muting dominance.

## Use case mapping

| Resource | Allocation rule | `slices` | `price` | `M` |
|---|---|---|---|---|
| Pizza order | A (with replacement) | slices in pizza | dollars | budget |
| Grant pool to multiple projects | A or B | project deliverables | dollars requested | pool size |
| Sole-source contract | B | 1 | 1 | 1 |
| Single-seat election | B | 1 | 1 | 1 |
| Multi-seat legislature | C (or B without replacement) | 1 | 1 | seats |

Across all cases the front end (approval + threshold) is identical. Only the allocation rule changes.

## Framing and prior art

The stochastic election variants (B) are politically difficult to defend — losing a 49% race to a draw feels wrong — but they are not novel and not without intellectual support. Akhil Amar's *Choosing Representatives by Lottery Voting* (Yale Law Journal, 1984) is the canonical legal-theory treatment; the broader sortition literature (e.g., Van Reybrouck, *Against Elections*) makes the political case. Jury selection is the everyday example: society already trusts a stochastic mechanism for some of the most consequential decisions an institution can make about an individual.

The deterministic weighted-representation variant (C) has prior art in the EU Council of Ministers and in the mathematical literature on weighted majority games (Banzhaf and Shapley-Shubik power indices), which can be used to verify that proportional *weight* yields proportional *power* in actual coalition dynamics.

The approval front end alone — without any lottery or weighting — already breaks two-party dynamics by giving broadly-acceptable candidates the weight that plurality voting discards. The lottery (A, B) or weighting (C) is a second, independent anti-majoritarian layer.

## Gaming considerations

- **Bullet voting** (approving only your top choice) is the main residual strategy under approval voting. The lottery softens it (no winner-take-all), so the cost of approving more options is low and the incentive to bullet-vote is correspondingly weak.
- **Bid inflation** is the main attack in the grants/contracts case: claim more `slices` than you can deliver, to inflate the `value` term. Conventional procurement has the same problem and the same answer — bid verification and audit.
- **Coalition coordination** is fair by construction: an aligned group is amplified in proportion to its actual size, no more.
- **Threshold** is the single biggest knob for the gaming/diversity tradeoff. Set too low it admits spoilers and noise; set too high it reverts toward majoritarian outcomes.

## Worked example (pizza)

Setup: 5 voters, budget `M = $30`, threshold = 2.

| Option | slices | price | value |
|---|---|---|---|
| Cheese | 8 | $12 | 0.667 |
| Pepperoni | 8 | $14 | 0.571 |
| Veggie | 6 | $15 | 0.400 |
| Hawaiian | 8 | $16 | 0.500 |

Approvals:

| Voter | Approves |
|---|---|
| Alice | Cheese, Pepperoni |
| Bob | Cheese, Pepperoni |
| Carol | Cheese, Veggie |
| Dave | Pepperoni, Hawaiian |
| Eve | Veggie |

### Step 1–2: votes and threshold

`votes`: Cheese = 3, Pepperoni = 3, Veggie = 2, Hawaiian = 1.

Hawaiian falls below threshold (1 < 2) and is eliminated.

### Step 3: allocation (rule A, with replacement)

**Draw 1.** `M = $30`. All remaining options fit. Weights `votes × value`:

- Cheese: 3 × 0.667 = 2.00
- Pepperoni: 3 × 0.571 = 1.71
- Veggie: 2 × 0.400 = 0.80

Total = 4.51 → P(Cheese) ≈ 44%, P(Pepperoni) ≈ 38%, P(Veggie) ≈ 18%.

Suppose **Cheese** is drawn. `M = $30 − $12 = $18`. Pool: 8 cheese slices.

**Draw 2.** `M = $18`. All three still fit. Same probabilities (with replacement).

Suppose **Pepperoni** is drawn. `M = $18 − $14 = $4`. Pool: 8 cheese + 8 pepperoni = 16 slices.

**Draw 3.** `M = $4 < min(price) = $12`. Loop terminates. $4 unspent.

Final order: 1 Cheese + 1 Pepperoni = 16 slices.

Notes from this run:
- The lottery means a different run could produce 2 Cheese, or 1 Cheese + 1 Veggie, etc. — each consistent with the approval distribution.
- Hawaiian had real support (Dave's approval) but did not clear the threshold and never entered the draws.
- $4 left unspent is a feature, not a bug: no remaining option fits the budget.

## License

See `LICENSE`.
