# Rosetta_attesoR

**Reversible fixed-point arithmetic with an exact ledger of what it costs to go backward.**

Every lossy operation throws away information. Rosetta records the *minimal witness* needed to undo it, so any transform runs forward and then peels back to the original value — exact, to the bit — and tells you precisely what that reversibility cost. No magic, no overclaiming: reversibility with a receipt.

The repository name is `Rosetta` reversed (`attesoR`), which is the whole point: the forward pass and the backward pass are the two halves of one bijection.

---

## What this is (and isn't)

This is a **research repository**. At its center is a small, fully exact, test-pinned kernel. Around that kernel is a larger body of exploratory code — reversible flows, coupling layers, a PCB-language lowering path, a manifold controller, and a JavaFX dashboard — that builds on the kernel to probe what reversible computation can be used for. The two have very different maturity levels, and this README keeps them clearly separated.

- **The kernel is exact and proven.** Bit-exact round trips, witness size equal to the map's log-determinant, an entropy coder that reaches the offset-entropy floor. These are pinned by tests, not asserted by prose.
- **The surrounding layers are research.** They compile and run, they're useful for exploration, but they are not production components and this README does not pretend otherwise.

---

## The core idea

A normal lossy operation — a rounding, a downscale, an overwrite — destroys information and is therefore irreversible. Rosetta makes the *same* operation reversible by keeping, at the boundary of the operation, exactly the information the operation would have destroyed and no more.

The substrate is a fixed-point integer lattice (`long` cells, resolution `S = 2^precisionBits`), **not** IEEE-754. That choice is load-bearing: integer add/subtract is an exact bijection, so a contraction's lost sub-grid remainder is recoverable from a small witnessed offset alone.

```
crossIn :  y = round(x * c)        offset_i = x_i - xmin(y_i)     // forward, lossy when c < 1
crossOut:  x = xmin(y) + offset    //  backward, exact
```

The witness's size is the transform's log-determinant — `-log2(c)` bits per contracted cell, and **zero** for `c >= 1` (expansions and volume-preserving relays are free to reverse). That is not a heuristic; it falls out of the arithmetic.

---

## The kernel API

Four small classes carry the entire guarantee.

### `FixedPointLattice`
The off-heap substrate. Cells are `JAVA_LONG`; resolution is `S = 2^precisionBits`. Backed by an automatic `Arena`, so callers manage no lifecycle.

```java
FixedPointLattice lattice = new FixedPointLattice(16);   // S = 2^16
MemorySegment cells = lattice.allocate(1024);            // 1024 long cells, zeroed
```

### `AffinePocket`
The crossing. `crossIn` runs the interior contraction `y = round(x * scale)` in place over every cell and returns the `Witness`; `crossOut` consumes the witness to reconstruct the input exactly. The scale is re-read on the way out, **never inverted**.

```java
Witness w = AffinePocket.crossIn(cells, 0.5);   // contract: lossy, witnessed
//  ... cells now hold the contracted values ...
AffinePocket.crossOut(cells, w);                // restore: bit-exact, guaranteed
```

`AffinePocket.collisions(y, c)` returns how many integers collapse onto `y` — the alphabet size of that cell's offset.

### `Witness`
The persistent record a lossy crossing leaves at its boundary. **Not** metadata and **not** the discarded data — it is the load-bearing information deficit required to invert the crossing.

- `bits()` — bits this witness actually stores (naive, per-crossing).
- `budgetBits()` — the conserved log-determinant floor the crossing is accountable for.
- `offsets()` — the per-cell payload, exposed for entropy coding.

### `WitnessCoder`
Mixed-radix arithmetic coding (the infinite-precision form of rANS) that packs the offset stream into `ceil(Σ log2 K_i)` bits — the offset-entropy floor — and unpacks it exactly. The residual gap from there down to the continuum log-determinant is the price of the integer lattice itself and is **not removable by any coder**. The README states this because the code does: it's a real limit, not coding slack.

```java
byte[] packed = WitnessCoder.encode(w.offsets(), radices);
long[] back    = WitnessCoder.decode(packed, radices);   // exact inverse
```

### `RosettaOnion`
A convenience wrapper that grows the kernel into "shells": `wrap(scale)` adds an outward layer and returns its witness bits; `peelToCore()` reverses every shell, in order, back to the exact center. Useful as a worked example of the kernel composed.

```java
RosettaOnion onion = new RosettaOnion(16);
onion.seedCore(987_654);
onion.wrap(2.0);     // expansion shell — free (0 witness bits)
onion.wrap(0.5);     // contraction shell — costs witness bits
assert onion.peelToCore() == 987_654;   // exact, through every shell
```

---

## φ — the irreversibility fraction

The kernel is bit-exact, so reversibility is never *in question*; what varies is its **price**. That price is captured by a single bounded number:

```
φ = Δ / R = 1 − B/R   ∈ [0, 1]
```

where `R` is the forward spend, `B` the part recoverable backward for free, and `Δ = R − B` the irreversible remainder. `φ = 0` is fully reversible (a contract-first valley — full refund); `φ = 1` is fully irreversible (an expand-first peak — full toll, no refund). It has a clean closed form, `φ = 1 − d`, where `d` is the valley depth of the scale-walk, so the relationship is an exact line, not a fit.

What the study corpus established (all numbers are measured, with CSVs to reproduce the graphs):

- **The distribution is a "saddle," not an arcsine law.** It has two genuine point-masses at the ends (`P(φ=0) ≈ 0.26`, `P(φ=1) ≈ 0.32` at light load) that an arcsine cannot have, plus a notched interior comb. KS gap to arcsine ≈ 0.30. The saddle is a light-load effect and dissolves under load.
- **Load is location; scale is granularity** — two orthogonal knobs. Mean φ depends on path length (load), nearly independent of exponent range (scale).
- **Power law:** `1 − meanφ ≈ 0.793 · len^(−0.359)`, R² ≈ 0.994 (pre-asymptotic; the exponent drifts mildly with load, and the README/report say so rather than pretending it's a perfect power law).

The full study lives under the docs tree as a 20-cell scaling sweep (scale × load) with per-cell histograms, scatter plots, rung CSVs, a master summary, and a cross-cell scaling report.

---

## Build & run

Requires **JDK 24** (the kernel uses the Foreign Function & Memory API) and **Maven 3.8+**.

```bash
mvn compile          # build
mvn test             # run the suite (this is where the guarantees are pinned)
```

> **Note on dependencies.** The full build depends on a local `PCB` language-runtime artifact (`PCB:PCB:0.0.1-SNAPSHOT`) and on JavaFX for the dashboard. The *kernel* (`FixedPointLattice`, `AffinePocket`, `Witness`, `WitnessCoder`, `RosettaOnion`) has no such dependency and is self-contained; the PCB/JavaFX requirements come from the exploratory layers and the dashboard entry point (`Main` → `RosettaDashboardApp`).

---

## What the tests verify

The kernel's guarantees are assertions in the suite, not claims in this file:

- **Bit-exact round trips** across mixed contractions and expansions, over tens of thousands of seeds (`RosettaOnionTest`, `ReversibleKernelTest`, `BoundaryAccountedCrossingTest`).
- **Witness size = log-determinant:** per-cell witness bits equal `log2(collisions)`, and equal the contraction amplitude exactly on dyadic scales (`HolonomyScalingStudyTest` — global `max|witnessBits − L| = 0`, zero mismatches).
- **Coder reaches the offset-entropy floor** and inverts exactly (`WitnessCoderTest`).
- **φ ∈ [0,1], measured = closed-form to 0.0000**, valley → 0, peak → 1 (`HolonomyIrreversibilityFractionTest`).
- The distribution studies, arcsine disproof, and scaling law (`HolonomyIrreversibilityStressTest`, `HolonomyArcsineFitTest`, `HolonomyScalingStudyTest`, `HolonomyScalingReportTest`).

---

## Honest limitations

- **Reversibility costs space, not cycles.** Keeping witnesses is *more* memory and bandwidth, not less. The win is exact reversibility at a known, minimal price — useful where you'd otherwise brute-force snapshot a reversible computation (checkpoint/restore, undo logs, reverse debugging, the backward pass of differentiable pipelines). It is not a way to make ordinary forward code run faster.
- **The continuum gap is real.** The coder reaches the lattice offset-entropy floor; the residual to the continuum log-determinant is the lattice's own price and cannot be coded away.
- **Only affine fixed-point crossings are proven.** Open paths, branching/joining graphs, and the broader flow/PCB/manifold machinery are research, not guarantees.

---

## Repository layout

```
src/main/java/com/rosetta/core/
  FixedPointLattice.java   AffinePocket.java   Witness.java   WitnessCoder.java   // the exact kernel
  RosettaOnion.java                                                              // composed example
  ReversibleKernel / ReversibleFlow / CouplingLayer / FlowTrainer / ...          // research: reversible flows
  Pcb*Lowering / VirtualPcbEnvironment / Manifold* / NativeManifold              // research: PCB integration
  RosettaDashboardApp / Main                                                     // JavaFX dashboard
src/test/java/com/rosetta/core/                                                  // the suite that pins it all
```

---

## License

TBD.
