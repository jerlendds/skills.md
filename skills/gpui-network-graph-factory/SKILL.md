---
name: gpui-network-graph-factory
description: Verification-first guidance for constructing and optimizing a Rust + GPUI network graph visualization library with agents, using immutable oracles, deterministic replay, bounded mutation surfaces, property/fuzz testing, and machine-gated correctness and performance acceptance.
---

# GPUI Network Graph Factory Skill

## Purpose

Guide agents constructing or optimizing a Rust + GPUI network-graph visualization library using a verification-first workflow.

The goal is not “agent writes code, human reviews PR.” The goal is:

> Agents search a bounded implementation space against machine-owned correctness and performance verdicts they cannot modify.

## Core Principle

For every subsystem, define:

1. **Reference semantics**
2. **Immutable oracle**
3. **Deterministic inputs**
4. **Replayable failures**
5. **Quantitative performance objective**
6. **Bounded agent-editable surface**

If these are missing, keep a human at the specification boundary.

## Repository Shape

Prefer separation like:

```text
graph-core/        graph model + invariants
graph-scene/       deterministic scene/geometry/culling
graph-layout/      layout traits + algorithms
graph-gpui/        GPUI integration/input/paint bridge
graph-reference/   slow/simple reference implementation
graph-verifier/    equivalence/property/fuzz/replay tests
graph-bench/       fixed workloads + regression thresholds
graph-agent/       task contracts + mutation boundaries
```

Agents may modify implementation crates. They must not modify reference semantics, oracle data, verifier acceptance rules, or benchmark thresholds unless explicitly tasked to change the specification.

Enforce this outside prompts, e.g. CI rejects unauthorized oracle/verifier changes.

## Verification Surfaces

### Graph semantics

Check machine-owned invariants:

- rendered node IDs == model node IDs
- rendered edge IDs == model edge IDs
- edge endpoints correspond to source/target nodes
- mutations never leave dangling IDs
- hit-testing resolves the expected entity
- selection/hover state remains valid after mutation

Use property-based and generated adversarial graphs:

- empty/singleton
- self-loops
- parallel edges
- disconnected components
- complete/star/path/bipartite graphs
- duplicate labels
- pathological coordinates
- very large sparse/dense graphs

Prefer reference-vs-optimized semantic equivalence over hand-written expected outputs.

### Coordinate transforms

Test algebraic properties:

```text
screen_to_world(world_to_screen(p)) ≈ p
T⁻¹(T(p)) ≈ p
pan(a); pan(b) == pan(a+b)
zoom(k); zoom(1/k) ≈ identity
zoom(anchor,k) preserves anchor
```

Stress extreme scales, coordinates, DPI, and viewport sizes.

Use distinct types for coordinate spaces where practical:

```text
WorldPoint
ScreenPoint
LayoutPoint
```

Do not collapse all spaces into the same raw point type unless conversion cost outweighs the prevented bug class.

### Determinism and replay

Core behavior should not implicitly depend on:

- wall clock
- OS randomness
- thread scheduling
- unordered map iteration
- ambient mutable globals
- GPU timing

Inject time, seed, viewport, DPI, and other environment inputs.

A failing run should reduce to a finite replay artifact:

```text
initial graph
+ seed
+ viewport
+ event sequence
= deterministic failure
```

Persist minimal failing traces.

### Visual correctness

Use layered verification:

1. semantic scene
2. canonical geometry
3. raster output

Prefer canonical primitive comparison before screenshot comparison:

```text
Node(id,bounds)
Edge(id,path)
Text(entity,bounds)
clip/z-order/visibility
```

This distinguishes layout/geometry bugs from font/raster/platform noise.

### Interaction correctness

Represent interaction as a replayable action DSL:

```rust
enum Action {
    Move(Point),
    MouseDown(Button),
    MouseUp(Button),
    Wheel(f32),
    KeyDown(Key),
    Resize(Size),
    Tick(Duration),
}
```

Fuzz event sequences and assert state invariants.

Examples:

- click(node) selects node
- click(background) clears selection
- pan/zoom never changes graph topology
- cancelled drag leaves valid state
- deleting hovered/selected entities leaves no dangling interaction state
- repeated hover/move operations are idempotent where expected

### Performance correctness

Use fixed workloads and immutable thresholds.

Track at least:

- frame construction p50/p95/p99
- paint p95
- hit-test latency
- graph mutation latency
- allocations/frame
- bytes/node and bytes/edge
- peak RSS
- viewport-query complexity

Representative workloads should span at least 1k, 10k, 100k, and where relevant 1M nodes.

Do not accept a candidate merely because tests pass. Require either:

- no meaningful regressions plus a target improvement, or
- an explicitly specified semantic/API change.

## Agent Task Contract

Do not assign vague tasks such as:

```text
Implement graph selection.
```

Assign bounded optimization/specification contracts:

```yaml
task: optimize_edge_hit_testing

editable:
  - graph-scene/src/hit_test/**
  - graph-gpui/src/edges.rs

hard_constraints:
  semantic_equivalence: true
  deterministic: true
  allocations_per_query: <= 1

objective:
  benchmark: edge_hit_test_100k
  minimize: p95_ns

regression_limits:
  peak_rss: +2%
  build_time: +3%
```

The agent may read all verifier output but must not redefine success.

## Acceptance Model

Prefer a verdict vector over a single `cargo test` result:

```text
compile
clippy
graph invariants
reference equivalence
property tests
deterministic replay
interaction tests
geometry/raster checks
fuzz count
sanitizer failures
performance metrics
memory metrics
```

Example acceptance policy:

```text
accept iff
  semantic checks == pass
  deterministic replay == pass
  sanitizer failures == 0
  fuzz budget reached
  regressions stay within limits
  and at least one target objective improves materially
```

## Rust-Specific Guidance

Push invalid states into types when the ergonomics remain acceptable.

Prefer strong IDs over raw indices:

```rust
struct NodeId(NonZeroU32);
struct EdgeId(NonZeroU32);
```

Consider typestate only where it enforces valuable phase invariants:

```text
Graph<UnlaidOut>
Graph<LaidOut>
Graph<Indexed>
```

Falsifier: if type machinery creates pervasive escape hatches, casts, or conversion boilerplate, simplify it.

## Automation Levels

### 1. Human-reviewed agents

```text
agent -> verifier -> PR -> human
```

Use while semantics are still unstable.

### 2. Machine-gated merge

```text
agent -> immutable verifier -> benchmark comparison -> merge/reject
```

Best for bounded implementation changes such as:

- spatial indexing
- culling
- cache layout
- allocation reduction
- tessellation
- incremental recomputation

### 3. Continuous implementation search

```text
task generator
    -> parallel candidate agents
    -> correctness filter
    -> benchmark tournament
    -> winner
    -> main
```

Use only when the verifier has very low false-negative probability.

## Human Boundary

Keep humans at the **specification/API boundary**, not the line-by-line implementation boundary.

Humans should own:

- semantic contracts
- public API constraints
- architectural boundaries
- benchmark objectives
- verifier/oracle changes

Agents may autonomously search implementation space beneath those constraints.

## Failure Modes

Reject or escalate when:

- the agent modifies its own oracle or acceptance threshold
- tests are generated only from the candidate implementation
- correctness depends on subjective review
- failures cannot be reproduced from a finite artifact
- benchmarks are noisy enough to reverse decisions
- platform-specific raster differences dominate visual verdicts
- optimization degrades API coherence, composability, or debuggability
- agent-editable and verifier-owned code are not mechanically separated

## Final Rule

Before automating a subsystem, ask:

> Which machine-owned verdict is this implementation allowed to fail, and can the implementation agent alter that verdict?

If no trustworthy, externalized verdict exists, the subsystem is not yet ready for closed-loop autonomous construction.
