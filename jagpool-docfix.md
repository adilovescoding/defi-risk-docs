# Final Delegation Weight Calculation

## Overview

JagPool allocates stake across regional validator sets to preserve Solana's geographic decentralization, minimize consensus-critical latency between cluster regions, and reduce the network's exposure to correlated regional failure or censorship. Regional partitioning of the validator set is what delivers the latency and fault-isolation benefits; within each partition, capital must still be routed toward the validators best serving delegators. JagPool therefore computes a **Base Validator Score** from yield and operational telemetry, then **normalizes and scales** that score within each region to produce the final delegation weight — the two-stage pipeline specified below.

---

## Section 1: Base Validator Score Calculation

### Metric Definitions

- **$\text{APY}(v)$** — the trailing-epoch annualized percentage yield delivered by validator $v$ to delegators. Treated as a **linear base multiplier**, not an exponentiated term, because APY differences between validators are typically small fractions (e.g., $6\%$ vs. $6.5\%$); exponentiating them alongside operational ratios would cause catastrophic underflow and erase meaningful yield differentiation.
- **$VC(v)$** — the **Vote Credits Ratio**: validator $v$'s vote credits earned in the trailing epoch, relative to the top-performing validator in its own region:
  $$
  VC(v) = \frac{\text{VoteCredits}(v)}{\displaystyle\max_{u \in R(v)} \text{VoteCredits}(u)}, \qquad VC(v) \in (0, 1]
  $$
- **$K(v)$** — the **Skip Rate**: the proportion of assigned leader slots validator $v$ failed to produce in the trailing epoch, $K(v) \in [0, 1]$. The score uses the **inverted liveness metric** $\big(1 - K(v)\big)$, so that skipping slots strictly *reduces* $S(v)$ — never rewards it.

### Real-Valued Specification

$$
S(v) = \text{APY}(v) \cdot \Big( VC(v) \cdot \big(1 - K(v)\big) \Big)^{\gamma}, \qquad \gamma = 5
$$

The operational performance ratio $VC(v) \cdot (1 - K(v))$ is isolated inside the exponent and raised to $\gamma = 5$ to aggressively separate elite, consistently-live validators from the field. $\text{APY}(v)$ is decoupled and applied as a post-exponent multiplier, preserving its resolution regardless of how steeply the operational term is punished.

> [!NOTE]
> **Worked example.** For $\text{APY}(v) = 6.5\%$, $VC(v) = 0.99$, $K(v) = 0.03$: the liveness-adjusted performance ratio is $0.99 \times 0.97 = 0.9603$, and $0.9603^{5} \approx 0.8151$. The base score is $S(v) = 0.065 \times 0.8151 \approx 0.05298$ — full yield resolution preserved, with the operational term still doing the heavy discriminating work.

### Fixed-Point Specification

Define $\text{APY}_{bps}(v) = \lfloor \text{APY}(v) \times 10^{4} \rfloor$ (basis points, so $6.5\% = 650$ bps), and let $\hat{VC}(v)$, $\hat{K}(v)$ be `ufixed64x9` integers scaled by the global precision constant $10^{9}$.

**Step 1a — Liveness-adjusted performance ratio:**

$$
\hat{\text{Perf}}(v) = \left\lfloor \frac{\hat{VC}(v) \cdot \big(10^{9} - \hat{K}(v)\big)}{10^{9}} \right\rfloor
$$

**Step 1b — Safe exponentiation via iterative rescaled multiplication:**

Raw fixed-point exponentiation ($\hat{\text{Perf}}(v)^{5}$ computed directly) overflows even `u128`, so the exponent is applied one multiply-and-rescale step at a time:

$$
\hat{x}_1 = \hat{\text{Perf}}(v), \qquad \hat{x}_{k+1} = \left\lfloor \frac{\hat{x}_k \cdot \hat{\text{Perf}}(v)}{10^{9}} \right\rfloor \quad \text{for } k = 1, \dots, \gamma - 1
$$

$$
\hat{\text{Perf}}^{\gamma}(v) := \hat{x}_{\gamma}
$$

**Step 1c — Decoupled APY multiplier:**

$$
\hat{S}(v) = \left\lfloor \frac{\text{APY}_{bps}(v) \cdot \hat{\text{Perf}}^{\gamma}(v)}{10^{4}} \right\rfloor
$$

$\hat{S}(v)$ is the final fixed-point Base Validator Score, `ufixed64x9`-scaled, and is the direct input to Section 2.

---

## Section 2: Regional Delegation Weight Normalization

Let $R(v)$ denote the set of all validators sharing $v$'s declared region, with $|R(v)| = n_r$. The final delegation weight is the region-normalized share of $\hat{S}(v)$, scaled by the protocol's baseline delegation multiplier of $5$.

**Real-valued specification:**

$$
W(v) = 5 \cdot \frac{S(v)}{\displaystyle\sum_{u \in R(v)} S(u)}, \qquad \sum_{v \in R} W(v) = 5
$$

**Fixed-point specification:**

$$
\hat{W}(v) = \left\lfloor \frac{5 \times 10^{9} \times \hat{S}(v)}{\displaystyle\sum_{u \in R(v)} \hat{S}(u)} \right\rfloor
$$

$\hat{W}(v)$ is an unsigned integer at $10^{-9}$ precision, where $\hat{W}(v) = 10^{9}$ corresponds to a floating-point weight of exactly $1.0$, bounded by the regional ceiling $\hat{W}(v) \le 5 \times 10^{9}$.

---

## Unified Parameter & Boundary Limits

| Symbol | On-Chain Type | Bounds / Constraints | Description |
|---|---|---|---|
| $\text{APY}(v)$ | reference (real-valued) | $\text{APY}(v) \ge 0$ | Trailing-epoch annualized percentage yield |
| $\text{APY}_{bps}(v)$ | `u32` | $0 \le \text{APY}_{bps}(v) \le 10{,}000$ (protocol ceiling $= 100\%$) | On-chain basis-point representation of $\text{APY}(v)$ |
| $VC(v)$ | reference (real-valued) | $VC(v) \in (0, 1]$ | Vote credits ratio relative to top regional performer |
| $\hat{VC}(v)$ | `u64` | $1 \le \hat{VC}(v) \le 10^{9}$ | Fixed-point $VC(v)$, scaled by $10^{9}$ |
| $K(v)$ | reference (real-valued) | $K(v) \in [0, 1]$ | Trailing-epoch leader-slot skip rate |
| $\hat{K}(v)$ | `u64` | $0 \le \hat{K}(v) \le 10^{9}$ | Fixed-point skip rate, scaled by $10^{9}$ |
| $\hat{\text{Perf}}(v)$ | `u64` | $0 \le \hat{\text{Perf}}(v) \le 10^{9}$ | Fixed-point liveness-adjusted performance ratio |
| $\gamma$ | `u8` constant | $\gamma = 5$ (protocol-fixed) | Exponent isolating operational performance from yield |
| $\hat{\text{Perf}}^{\gamma}(v)$ | `u64` | $0 \le \hat{\text{Perf}}^{\gamma}(v) \le 10^{9}$ | Fixed-point exponentiated performance ratio |
| $\hat{S}(v)$ | `u128` (accum.) → `u64` (stored) | $\hat{S}(v) \ge 0$ | Final fixed-point Base Validator Score |
| $R(v)$, $n_r$ | `Vec<Pubkey>`, `u32` | $n_r \ge 1$ | Regional validator set and its cardinality |
| `PRECISION` | `u64` constant | $10^{9}$ (fixed) | Global fixed-point scaling precision |
| `SCALE_FACTOR` | `u8` constant | $5$ (fixed) | Baseline regional delegation multiplier |
| $\hat{W}(v)$ | `u64` | $0 \le \hat{W}(v) \le 5 \times 10^{9}$ | Final scaled, normalized regional delegation weight |
| $D$ | `u32` | $0 \le D < n_r$ | Truncation dust redistributed via apportionment pass |

---

## Runtime Optimization Callout

> [!CAUTION]
> **Precision, Overflow, and Exponentiation Safety**
> - **Never call `pow()` directly** on a $10^{9}$-scaled fixed-point integer with $\gamma = 5$. A naive $\hat{\text{Perf}}(v)^{5}$ can reach $(10^{9})^{5} = 10^{45}$, far exceeding `u128::MAX` ($\approx 3.4 \times 10^{38}$). Always use the iterative rescale-after-multiply method in Step 1b, which bounds every intermediate product to at most $10^{18}$ — safely inside `u128`, and even inside `u64` for most operand ranges.
> - **Multiply before dividing, always.** Pre-scaling $\hat{VC}(v)$ or $\hat{K}(v)$ down before multiplying silently truncates precision at every step and compounds error across all five exponentiation rounds. Perform the full product first, then divide once.
> - **Widen before you accumulate.** Every intermediate product (Steps 1a, 1b, 1c, and the Section 2 normalization) must be computed in `u128` before being downcast to `u64` on final assignment. Use checked arithmetic (`checked_mul`, `checked_div`) and reject the instruction on `None` rather than saturating or wrapping silently.
> - **Keep $\text{APY}_{bps}(v)$ decoupled from the exponent.** Folding APY into the exponentiated term collapses small, healthy yield variances toward zero (a standalone $6\%$ APY term raised to the 5th power underflows to $\sim 10^{-7}$), erasing meaningful yield differentiation between validators. Applying it as a post-exponent multiplier preserves full yield resolution.

```rust
// Section 1 — Base Validator Score (per validator, per epoch)
const PRECISION: u128 = 1_000_000_000; // 10^9
const GAMMA: u32 = 5;

fn compute_perf_pow(perf_hat: u64) -> u64 {
    let perf = perf_hat as u128;
    let mut x = perf;
    for _ in 1..GAMMA {
        x = (x * perf) / PRECISION; // rescale after every multiply — no raw pow()
    }
    x as u64
}

fn compute_base_score(apy_bps: u32, vc_hat: u64, skip_hat: u64) -> u64 {
    let perf_hat = ((vc_hat as u128) * (PRECISION - skip_hat as u128) / PRECISION) as u64;
    let perf_pow_hat = compute_perf_pow(perf_hat);
    ((apy_bps as u128 * perf_pow_hat as u128) / 10_000) as u64 // decoupled APY multiplier
}

// Section 2 — Regional normalization + scaling (per region, per epoch)
const SCALE_FACTOR: u128 = 5;

let target: u128 = SCALE_FACTOR * PRECISION;
let sum_s_hat: u128 = region.iter().map(|v| v.s_hat as u128).sum();

let mut allocated: u128 = 0;
let mut remainders: Vec<(Pubkey, u128, u64)> = Vec::with_capacity(region.len());

for v in region.iter() {
    let numerator = target * (v.s_hat as u128);
    let w_hat = numerator / sum_s_hat;          // floor division
    let remainder = numerator % sum_s_hat;
    allocated += w_hat;
    remainders.push((v.pubkey, remainder, w_hat as u64));
}

let mut dust = (target - allocated) as usize;
remainders.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0))); // remainder desc, pubkey asc

for (_, _, w_hat) in remainders.iter_mut().take(dust) {
    *w_hat += 1;
}
// invariant: sum(w_hat) == target, no overflow, deterministic across all nodes
```
