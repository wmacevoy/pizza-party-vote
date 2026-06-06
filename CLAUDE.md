# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Pre-implementation. The repo currently contains only `README.md` (the spec), `LICENSE`, and `.gitignore` — no source code, build system, tests, or chosen language. If asked to implement something, confirm the language/runtime with the user before scaffolding.

## What this project is

A voting mechanism for repeated games with discrete outcomes, optimizing for **average satisfaction** (not majority preference). The pizza framing in `README.md` is a worked example; the intended real-world targets are large grants/contracts and elections — specifically, breaking simple-majority dominance and spreading awards across a broader set of participants.

## Mechanism shape

The spec is a **framework**, not a single algorithm. There is one shared front end and three back-end allocation rules. When implementing, expect the same code to drive all three modes — don't fork the codebase by use case.

### Shared front end

1. **Approval voting.** Each voter sets `preference(voter, option) ∈ {0, 1}`. No ranking, no scoring.
2. **Quorum filter.** `votes(option) = Σ preference(voter, option)`. Options below a parameterized quorum threshold are zeroed out.

### Allocation rules (pick one per problem)

- **A. Stochastic with replacement.** While `M ≥ min(price)`: draw an option with `P ∝ votes × value` from candidates with `price ≤ M`, deduct `price`. Same option may be drawn repeatedly. Use case: pizza order, grant pools where the same vendor may win multiple awards.
- **B. Stochastic without replacement.** Same as A, but selected options are removed before the next draw. Use case: sole-source contracts, single-seat elections, grants restricted to one award per recipient.
- **C. Deterministic weighted top-M.** Take the top `M` options by `votes`. Seat each with voting weight proportional to its `votes`. Use case: multi-seat representative bodies where randomness is politically unacceptable. Equivalent in expectation to B over many roll-call votes.

### Optional distribution phase

Only meaningful when the resource is divisible *among the voters themselves* (the pizza case). Do **not** include it for grants, contracts, or elections. Rank voters by `expect_value = mean(value(approved options))`, ties random, round-robin pick. Use original `value` for ranking even if the option was eliminated by quorum.

## Implementation notes

- **Formula:** `P ∝ votes × value` where `value = slices / price`. The multiplication is intentional — it rewards both popular and efficient options. An earlier draft of the spec wrote division; if you see `/` in any comment or code, it's a stale bug.
- **RNG.** Both stochastic modes need a **seedable** RNG so runs are reproducible for tests. Don't reach for the language's default global RNG.
- **Integer arithmetic.** Budgets, prices, slices, and seats are integers in the spec. Prefer integer types end-to-end to avoid rounding surprises in the budget loop's termination check.
- **Quorum is a parameter.** No universal best rule (top-K, fixed fraction, fixed integer are all valid). Surface it in the API; don't hardcode.
- **Mode C procedural shifts.** If implementing the weighted-representation mode, "majority present" and quorum-to-conduct-business become weight-based, not head-count-based. Optional cap and vote transforms (sqrt/log) are mentioned in the README as knobs to mute concentration.
- **Mode B termination.** Loop ends when either `M` is exhausted or no remaining option has `price ≤ M` — both conditions matter.

## Sources / framing to know

- Stochastic election variants are politically hard but have legal-theory backing (Amar, *Choosing Representatives by Lottery Voting*, Yale L.J. 1984) and sortition-tradition support (Van Reybrouck, *Against Elections*).
- Weighted-representation variant has prior art in the EU Council of Ministers and the weighted majority games literature (Banzhaf, Shapley-Shubik power indices).
- The approval front end alone — independent of any lottery or weighting — is the primary anti-dichotomy mechanism. The allocation rule is a second, independent layer.
