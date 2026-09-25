---
name: scalable-network-graph-visualization
description: >-
  Use this skill when designing, implementing, profiling, reviewing, or optimizing
  an interactive network-graph visualization library, especially for large graphs,
  WebGPU/GPU rendering, force-directed or multilevel layout, semantic zoom,
  level-of-detail, graph aggregation, edge density, picking, streaming, or
  out-of-core graph exploration. The governing objective is stable interaction
  latency and faithful task-relevant information, not drawing every primitive.
---

# Scalable Network-Graph Visualization

Remember: system, developer, and user instructions take precedence.

## Primary goal

Build graph visualization systems whose per-frame work is bounded primarily by the
**visible/informative working set**, not directly by total `|V| + |E|`.

For every design or optimization task:

1. Identify the actual bottleneck before changing architecture.
2. Preserve graph semantics required by the user's task.
3. Maintain a configurable frame-time budget.
4. Degrade representation quality predictably before missing interaction deadlines.
5. Make expensive refinement incremental and reversible.
6. Measure p50/p95/p99 latency, not only average FPS.
7. Treat layout, rendering, interaction, streaming, and queryability as separate performance domains.

Do **not** define “large graph” only by node or edge count. A graph is large relative
to a device and task when full-detail processing, submission, shading, memory,
interaction, or layout cannot satisfy the required latency and visual fidelity.

---

# 1. First diagnose the workload

Before proposing optimizations, classify the bottleneck.

Measure independently:

- main-thread / host-language graph processing;
- draw/dispatch submission overhead;
- GPU vertex/fragment/raster work;
- GPU compute work;
- CPU↔GPU buffer uploads;
- GPU↔CPU readback stalls;
- edge overdraw / blending pressure;
- label rendering and collision work;
- layout iteration cost;
- hierarchy/index rebuild cost;
- picking latency;
- tile/network/storage latency;
- memory footprint and allocation churn;
- garbage collection / allocator pauses;
- cache hit rate and LOD churn.

Do not infer the bottleneck from `|V|` or `|E|` alone.

### Minimum instrumentation

Expose counters/timers for:

```text
frame.cpu_ms
frame.gpu_ms
frame.present_interval_ms
frame.p50_ms
frame.p95_ms
frame.p99_ms

render.visible_nodes
render.visible_edges
render.visible_labels
render.draw_calls
render.edge_samples
render.screen_coverage

compute.layout_ms
compute.lod_ms
compute.compaction_ms
compute.bundle_ms

transfer.upload_bytes
transfer.upload_ms
transfer.readback_ms

lod.refines
lod.coarsens
lod.frontier_size
lod.representation_counts[*]

stream.cache_hit_ratio
stream.requested_tiles
stream.resident_tiles
stream.bytes_per_second

interaction.pick_ms
interaction.input_to_visual_ms
```

Prefer GPU timestamp queries when available. Fall back to coarse CPU-side timing only
when the API/device cannot expose reliable GPU timings.

---

# 2. Select the simplest architecture that can satisfy the target

Maintain three viable architectures until measurements discriminate among them.

## Architecture A — flat GPU-resident graph

Use when topology and attributes fit comfortably in GPU memory and full or nearly
full graph rendering can remain inside the frame budget.

Characteristics:

- topology stored once in GPU storage buffers;
- node positions in GPU-readable buffers;
- edges store endpoint indices, not duplicated endpoint positions;
- node and edge geometry generated through instancing or compute-driven expansion;
- layout compute writes the same position buffers consumed by rendering;
- little or no CPU work proportional to `|V| + |E|` each frame.

This is the default starting architecture. Do not build a hierarchy until profiling
shows a need.

## Architecture B — hierarchical / multiresolution graph

Use when the full graph fits in memory but full-detail traversal or rendering does
not fit the frame budget, or when dense overplotting makes full detail meaningless.

Characteristics:

- cluster tree or DAG;
- active LOD frontier;
- several representations per cluster;
- screen-space/error/semantic refinement criteria;
- temporal coherence and hysteresis;
- hard output caps;
- GPU compaction and indirect rendering where practical.

## Architecture C — tiled / streamed / out-of-core graph

Use when the graph, derived representations, or attributes cannot stay resident, or
when a server/offline preprocessing pipeline can cheaply build multiscale data.

Characteristics:

- fixed-size multiresolution tiles/chunks;
- GPU-resident working-set cache;
- coarse parent representation available before child detail;
- background fetch/decode/upload;
- camera-velocity-aware prefetch;
- bounded upload/refinement budget per frame;
- cancellation/versioning for stale requests.

### Escalation rule

Prefer:

```text
A -> A + packetization -> B -> B + streaming -> C
```

Do not start with C because it sounds scalable. Complexity is itself a performance
and correctness cost.

---

# 3. Performance contract

A renderer must have an explicit target refresh rate or frame budget.

For target refresh `f`:

```text
T_frame = 1000 / f  milliseconds
```

Examples:

```text
60 Hz -> 16.67 ms
30 Hz -> 33.33 ms
120 Hz -> 8.33 ms
```

Do not budget the full interval. Reserve headroom for OS/browser scheduling,
compositor variance, sporadic uploads, and interaction spikes.

A useful initial project policy is:

```text
operating_budget = 0.70 .. 0.80 * T_frame
```

This is a project default, not a universal constant. Recalibrate on reference devices.

### Separate budgets

Maintain at least:

```text
render_budget_ms
compute_budget_ms
upload_budget_bytes
lod_update_budget_ms
layout_budget_ms
```

Do not let a layout iteration consume an unbounded fraction of a render frame.

### Degradation order

When predicted work exceeds budget, reduce low-value work before violating the
interaction deadline. A reasonable default ordering is:

1. decorative labels;
2. non-focused edge detail;
3. far-field node detail;
4. bundling/refinement iterations;
5. layout iterations;
6. optional visual effects.

Preserve, where possible:

- hovered or selected objects;
- direct interaction targets;
- requested paths/neighborhoods;
- focus/context landmarks;
- semantic state such as selection, filtering, and search results.

“No geometry” is a valid LOD representation for sufficiently low-benefit objects.

---

# 4. GPU data model

Prefer structure-of-arrays or compact packed buffers over object-per-node / object-per-edge
representations in the hot path.

A baseline graph representation:

```text
NodePositions:   array<vec2<f32>> or array<vec3<f32>>
NodeFlags:       array<u32>
NodeStyleIndex:  array<u32>
NodeCluster:     array<u32>

Edges:           array<vec2<u32>>   // source_index, target_index
EdgeFlags:       array<u32>
EdgeStyleIndex:  array<u32>
```

For algorithms needing adjacency, keep a CSR/CSC-like structure separately:

```text
RowOffsets: array<u32>
Neighbors:  array<u32>
```

Do not force the rendering edge list and algorithmic adjacency structure to be the
same representation when their access patterns differ.

### Stable identity

A node or edge must retain stable semantic identity across:

- layout updates;
- LOD changes;
- clustering;
- tile eviction/reload;
- compaction;
- filtering;
- selection.

Never use transient buffer offsets as the only public identity.

### Buffer rules

- Keep immutable topology resident whenever possible.
- Update only mutable fields.
- Avoid rebuilding edge endpoint positions on the CPU after node motion.
- Let the edge shader dereference source/target node positions.
- Reuse pipelines, bind groups, descriptors, and buffers where possible.
- Chunk data when storage-buffer size or index-range limits require it.
- Treat device limits as runtime inputs, not compile-time assumptions.
- Avoid synchronous GPU readback in the interaction loop.

---

# 5. Packetize work: graphlets / edgelets

Borrow the _work granularity_ idea from meshlets without assuming mesh-shader support.

Create GPU-friendly packets containing modest fixed or bounded amounts of graph
geometry. Tune packet size empirically.

Example descriptor:

```text
GraphPacket {
    bounds_screen_or_world
    node_range
    edge_range
    node_count
    edge_count
    cluster_id
    lod_representation_refs
    aggregate_weight
    semantic_importance
    version
}
```

Packets should be grouped primarily by:

1. spatial locality;
2. semantic/community locality when useful;
3. stable topology locality;
4. memory coherence.

Morton/Hilbert ordering is worth testing where spatial locality dominates.

Do not claim packetization is automatically faster. Falsify it against flat
instancing on representative hardware. Packet metadata, compaction, and dispatch
cost can dominate for small graphs.

---

# 6. Representation ladder

Every hierarchical region/cluster may have several valid render representations.

A practical ladder:

```text
R0  hidden
R1  aggregate glyph / heat-density splat
R2  aggregate inter-cluster connectivity
R3  bundled or sampled edges + aggregate nodes
R4  individual nodes + simplified individual edges
R5  full focus detail + labels / interaction affordances
```

These are not required to form a single strict global sequence. Different graph
regions may use different representation families simultaneously.

For very dense graphs, also consider an **alternate representation**, not merely a
lower node-link LOD:

- adjacency matrix;
- density field;
- community map;
- flow map;
- path/neighborhood-only view.

Node-link diagrams are not semantically privileged when edge density exceeds useful
visual discriminability.

---

# 7. Predictive cost/benefit LOD scheduler

Prefer a predictive frame-budget allocator over a purely reactive “FPS dropped, now
reduce quality” controller.

For each candidate representation change, estimate:

```text
DeltaCost
DeltaBenefit
Value = DeltaBenefit / max(DeltaCost, epsilon)
```

Start from a coarse feasible representation set. Apply the highest-value upgrades
until another upgrade would exceed the current budget.

This is a practical greedy approximation to a multiple-choice resource allocation
problem.

### Benefit model

A starting model may include:

```text
benefit =
    w_area      * projected_screen_area
  + w_focus     * focus_weight
  + w_select    * selection_weight
  + w_semantic  * semantic_importance
  + w_change    * structural_change_or_novelty
  + w_task      * task_relevance
  - w_motion    * camera_motion_penalty
  - w_switch    * representation_switch_penalty
```

The exact form must be benchmarked and user-task dependent.

Do not use screen area alone. A tiny selected node may be more valuable than a huge
background cluster.

### Cost model

Estimate cost from observed timings, not primitive counts alone.

Maintain device/session-specific online estimates, for example exponentially weighted
moving averages grouped by:

- representation type;
- packet size;
- edge length/sampling regime;
- shader/pipeline;
- viewport resolution;
- device class.

Primitive count is a useful feature, not a sufficient model.

### Hysteresis

Use separate refinement/coarsening thresholds:

```text
refine if error > E_high
coarsen if error < E_low
where E_low < E_high
```

Optionally require the condition for multiple frames.

Do not let objects oscillate between LODs due to one-pixel camera noise.

---

# 8. View-dependent refinement

Maintain an **active frontier** in the cluster hierarchy rather than re-traversing the
entire graph from roots to leaves each frame.

Use previous-frame frontier state as the starting point.

Per active cluster/packet, compute some combination of:

- viewport intersection;
- projected area;
- projected node density;
- projected edge density;
- representation error;
- focus distance;
- selection state;
- semantic importance;
- data availability;
- expected refinement cost.

Refine only when the expected information gain justifies the cost.

### Hard caps

Always support configurable hard ceilings such as:

```text
max_visible_nodes
max_visible_edges
max_visible_labels
max_edge_samples
max_packets
max_lod_changes_per_frame
```

Hard caps are safety rails, not the primary quality policy.

### Amortization

If frontier adaptation itself becomes expensive, distribute it across frames.

During rapid pan/zoom:

- spend less time refining;
- prefer coarse stable representations;
- lower edge sampling/bundling quality;
- delay fine labels;
- restore detail after motion decelerates.

The renderer should become _cheaper_, not more expensive, during violent camera motion.

---

# 9. Screen-space aggregation

For dense overview views, the limiting resource is often **pixels**, not graph elements.

A transferable method from graph LOD work:

1. project nodes into a screen-space accumulation buffer;
2. count/weight contributions per pixel or tile;
3. create density kernels/splats only for occupied pixels/tiles;
4. aggregate edges using the node density field, cluster structure, or screen bins;
5. render detail only where projected information density permits discrimination.

This changes work from “one visual primitive per graph entity” toward work bounded by
viewport resolution and visible information density.

### Important limitation

Pure image-space aggregation can destroy provenance.

If the application must answer:

> Which exact nodes/edges contributed to this aggregate?

then retain one of:

- object-space aggregate membership;
- tile/cluster membership index;
- secondary ID lists;
- lazy query support back to canonical graph storage.

Do not pretend an image-space density blob is equivalent to a selectable graph object.

### False-structure control

Density kernels and edge bundling can create visually compelling structures that are
not robust properties of the graph.

Test aggregation under:

- uniform random positions;
- varying kernel bandwidth;
- rescaled viewport;
- randomized edge endpoints preserving degree distribution;
- alternate cluster hierarchies.

If a pattern disappears under small reasonable parameter changes, expose that
instability rather than presenting it as graph structure.

---

# 10. Edge rendering strategy

For dense node-link views, edges often dominate useful work and overdraw.

Do not assume opaque-geometry techniques such as hierarchical Z occlusion will solve
this problem. Transparent/subpixel graph edges usually **do not occlude each other in
a useful way**.

Prefer, depending on task:

- packet-level viewport rejection;
- semantic edge filtering;
- inter-cluster aggregation;
- screen-space density accumulation;
- edge sampling;
- edge bundling;
- selected-neighborhood emphasis;
- path-only refinement;
- alternate matrix/density views.

### Sampling

When an algorithm samples points along edges, sample in projected/screen space when
visual quality is the objective.

CUBu found roughly one sample per ten pixels useful in its tested bundling workloads.
Treat this only as a starting benchmark point, **not** a universal constant.

Adaptive alternatives should test:

```text
samples ~= ceil(projected_edge_length / sample_spacing_px)
```

with caps and lower sampling during motion.

### Gather before scatter

For density maps and bundling, prefer gather-style computations when they reduce
high-contention writes and atomics.

Benchmark both; hardware and workload can reverse the result.

### Fidelity rule

Selected edges, focused paths, or analytically important connections should be able
to bypass aggregation and render individually even when the background is aggregated.

---

# 11. Multiresolution tiles and graph clipmaps

For very large or remote graphs, use a multiresolution tile pyramid or equivalent
hierarchical chunk store.

A tile should contain enough metadata to render a valid coarse representation without
requesting descendants.

Typical tile contents:

```text
bounds
scale/level
aggregate node count / weight
aggregate edge count / flow
cluster summaries
coarse geometry
child references
stable semantic IDs or query handles
version/hash
```

### Working-set cache

Maintain a fixed or bounded resident pool.

On cache miss:

1. render the nearest available coarse parent/impostor;
2. request missing detail asynchronously;
3. decode off the interaction-critical thread where possible;
4. upload within a bounded per-frame byte/time budget;
5. swap in refined content without blocking the camera.

Never show a blank region merely because fine detail is unavailable if a coarse
representation exists.

### Prefetch

Use camera motion to prioritize likely future tiles:

```text
priority = f(distance_to_view,
             projected_scale,
             camera_velocity,
             zoom_velocity,
             semantic_focus,
             cache_state)
```

Cancel or deprioritize stale requests when the camera direction changes.

### Motion-dependent degradation

When the user moves faster than detail can stream/update:

- preserve coarse levels;
- stop chasing fine tiles that will immediately leave view;
- reduce update bandwidth spent on transient regions;
- converge toward detail after motion slows.

This is a feature, not a failure.

---

# 12. Layout architecture

Treat **layout throughput** and **render throughput** as separate measurements.

Support at least these modes conceptually:

1. static/precomputed layout;
2. local/incremental relaxation;
3. global multilevel layout;
4. GPU force-directed layout.

For very large graphs, static or server-computed coarse layouts may be superior to
continuous force simulation.

### Force-directed layout

Avoid exact all-pairs repulsion.

Use one of:

- Barnes-Hut / quadtree approximation;
- fast multipole-like approximation;
- multilevel graph hierarchy;
- community-level approximation;
- GPU spatial hierarchy.

Do not rebuild expensive irregular structures on the CPU every animation frame unless
profiling proves the cost acceptable.

### GPU caveat

“Put layout on the GPU” is not itself a performance argument.

GraphWaGu shows strong gains on capable GPUs, but its reported low-end integrated-GPU
case includes regimes where CPU-side D3-force outperformed its GPU layout because
GPU/compute overhead and hardware capability matter.

Therefore benchmark at least:

- high-end discrete GPU;
- common integrated GPU;
- software/fallback or CPU mode if supported.

### Decouple render and simulation rates

A heavy layout need not iterate at display refresh frequency.

For example:

```text
render: 60 Hz
layout: adaptive 10-30 Hz
```

Interpolate visual positions if necessary.

Physics/layout results must not change merely because the display refresh rate changes;
use explicit time steps or convergence control.

### Layout quality metrics

Benchmark more than iteration time. Depending on algorithm, track:

- stress;
- neighborhood preservation;
- edge-length distribution;
- overlap;
- crossings or angular resolution where relevant;
- temporal stability;
- cluster separation;
- task success.

A layout that converges quickly to a misleading or unstable configuration is not a
performance success.

---

# 13. WebGPU-oriented rendering pipeline

A strong baseline pipeline is:

```text
canonical graph storage
       |
       v
GPU topology buffers --------------------+
       |                                  |
       v                                  |
GPU layout / transform compute            |
       | writes                            |
       v                                  |
node position buffers <-------------------+
       |
       +--> LOD classification / packet compaction
       |        |
       |        +--> visible node/edge packet buffers
       |
       +--> node render pass
       |
       +--> edge render pass
       |
       +--> optional density/bundle passes
       |
       +--> optional pick-ID pass
```

### Rendering rules

- Prefer one/few draws per representation class over one draw per entity.
- Use instancing or storage-buffer indexing.
- Generate edge geometry from endpoint indices on GPU.
- Keep pipeline/bind-group configuration stable across frames.
- Use compute compaction and indirect draw where supported and beneficial.
- Avoid CPU loops that rewrite every edge because node positions changed.
- Reuse temporary buffers through pools/ring allocators.
- Avoid per-frame shader/pipeline creation.
- Minimize state switches, but do not fuse unrelated passes when fusion increases
  bandwidth or synchronization cost.

### Lines

Do not rely on platform-specific wide-line rasterization behavior.

For consistent styled edges, consider generating camera-facing quads/triangles from
line segments or using screen-space procedural edge expansion.

### Precision

For huge world coordinates, test floating-point precision at extreme zoom.

Potential mitigations:

- camera-relative coordinates;
- tile-local coordinates + origin;
- high/low coordinate split;
- CPU `f64` canonical positions with GPU-local `f32` transforms.

---

# 14. Picking and interaction

Interaction latency is a first-class benchmark.

### Picking options

Maintain multiple paths:

- ID render target for rendered objects;
- CPU spatial index over current visible working set;
- coarse tile/cluster hit test followed by refinement;
- GPU compute hit test for large visible sets.

Avoid synchronous readback on every pointer event.

If using an ID texture:

- issue asynchronous readback where the environment permits;
- tolerate one-frame hover latency if necessary;
- persist the last valid hover state until a new result arrives;
- keep selection independent of transient LOD representation.

### High-degree pathology

Selecting a node with degree `10^5` or `10^6` must not automatically force every
incident edge into full-detail rendering.

Possible policies:

- show top-k by task-specific weight;
- aggregate the remainder;
- progressively reveal;
- use histogram/density summaries;
- render paths on demand;
- expose a “show all” operation with explicit cost.

### Labels

Labels require their own LOD and collision budget.

Use hard caps, priority queues, and semantic ranking. Do not create DOM elements for
all graph labels in a large graph.

---

# 15. Semantic LOD

Geometric LOD alone is insufficient for analytical graphs.

A hierarchy should, where possible, preserve quantities meaningful to user tasks:

- community membership;
- edge direction;
- edge weight/flow;
- node/edge type;
- temporal interval;
- provenance;
- centrality or other declared importance metrics;
- filter state;
- selected/search membership.

When building automatic clusters, validate that the hierarchy is navigable and
analytically useful. A fast hierarchy that combines unrelated entities can make
multiscale navigation worse than a slower representation.

### Aggregate conservation tests

Where aggregation claims to summarize counts or weights, verify invariants such as:

```text
parent.node_count == sum(child.node_count)
parent.edge_weight == sum(represented_child_edge_weight)
```

Allow exceptions only when sampling/filtering is explicit in metadata.

---

# 16. Do not transfer game-rendering techniques blindly

## Techniques that transfer well

- hierarchical LOD;
- screen-space error / projected benefit;
- frame-budgeted quality selection;
- temporal coherence;
- hysteresis;
- packetized work;
- GPU-driven compaction;
- fixed-size working sets;
- multiresolution tiles;
- coarse impostors;
- camera-motion-aware streaming;
- bounded update work;
- graceful degradation.

## Techniques that transfer only partially

### Occlusion culling / Hi-Z

Useful for opaque 3D node glyphs, 3D terrain-like structures, or thick opaque regions.
Usually weak for transparent 2D edge clouds because edges do not remove enough later
work through occlusion.

Graph analogue: **density termination / semantic suppression / aggregation**, not just
hidden-surface rejection.

### Polygon simplification

Mesh geometric error has a direct surface interpretation. Graph simplification can
alter topology or analytical meaning.

Graph analogue must separate:

- visual geometry simplification;
- topology simplification;
- semantic aggregation.

Do not silently replace one with another.

### Edge bundling

Bundling can reduce clutter and work but changes apparent connectivity geometry.
Always preserve access to canonical edges and allow important edges/paths to bypass the bundle.

---

# 17. Known failure modes

Reject or challenge designs exhibiting these patterns unless measurements justify them.

| Failure                                              | Why it fails                                    | Preferred correction                                 |
| ---------------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------- |
| CPU object per node/edge in hot render path          | allocation, pointer chasing, GC/cache pressure  | compact typed/SoA buffers                            |
| one draw call per node/edge                          | submission/state overhead                       | instancing, packets, indirect draws                  |
| recompute edge endpoint geometry on CPU after layout | `O(E)` host work + upload                       | store endpoint IDs, dereference positions on GPU     |
| render every edge at overview scale                  | raster/overdraw dominates; visually meaningless | density/aggregation/filtering                        |
| rebuild full hierarchy every frame                   | update work scales with total graph             | temporal frontier + incremental update               |
| purely reactive FPS thresholds                       | overshoot/oscillation under abrupt complexity   | predictive cost/benefit + measured calibration       |
| no hysteresis                                        | LOD thrashing/popping                           | separate refine/coarsen thresholds                   |
| synchronous GPU readback for hover                   | CPU/GPU pipeline stall                          | async or CPU-visible-set picking                     |
| unbounded label layout                               | labels become dominant                          | hard cap + semantic priority                         |
| tile miss produces blank view                        | interaction appears broken                      | coarse parent/impostor fallback                      |
| prefetch ignores camera velocity                     | cache churn / wasted bandwidth                  | direction/zoom-aware priority                        |
| hierarchy ignores user semantics                     | fast but analytically misleading                | semantic validation / alternate hierarchies          |
| bundling shown as raw topology                       | perceptual false inference                      | explicit aggregate semantics + inspectable originals |
| “GPU always wins” assumption                         | integrated devices / small jobs may lose        | device-aware crossover benchmark                     |
| performance measured only by average FPS             | hides stutters/input failures                   | p50/p95/p99 + input latency                          |

---

# 18. Pathological test graphs

Every serious optimization should be tested on adversarial graph shapes, not only
social/community graphs.

Minimum synthetic suite:

### Empty/minimal

```text
0 nodes / 0 edges
1 node
2 nodes / 1 edge
self-loop
parallel duplicate edges
```

### Uniform random screen distribution

A worst case for screen-space aggregation because many pixels may become seed points
and spatial locality may be poor.

### Dense Erdős-Rényi

Tests overdraw, edge count, and failure of assumptions about community structure.

### Complete / near-complete subgraph

Tests whether the renderer understands that individual edge rendering is futile at
most zoom levels.

### Complete bipartite

Creates severe crossing density with simple topology.

### Star / supernode

One node with enormous degree. Test selecting, dragging, and focusing the hub.

### Barabási-Albert

Skewed degree distribution and hubs.

### Grid/geometric graph

Strong spatial locality. Useful counterpoint to random topology.

### Many disconnected components

Tests hierarchy and bounds overhead.

### Moving/churning topology

Insert/delete/update edges during camera interaction. Tests invalidation and buffer
fragmentation assumptions.

### Memory boundary

Graph size just below and just above:

- single-buffer device limit;
- practical GPU-memory budget;
- tile-cache capacity.

### Display boundary

Test:

- tiny viewport;
- 4K/high-DPR viewport;
- extreme zoom-out;
- extreme zoom-in;
- rapid oscillating zoom;
- fast pan reversal.

---

# 19. Benchmark protocol

Benchmark a matrix, not a single hero demo.

## Graph sizes

Use logarithmic scaling appropriate to the library target, for example:

```text
1k
10k
100k
1M
10M nodes
```

and multiple edge/node ratios such as:

```text
1x
5x
10x
50x
100x
```

Stop where memory/device constraints make a configuration invalid; report that
boundary explicitly.

## View states

For each graph test:

1. full overview;
2. zoomed sparse local region;
3. worst dense region;
4. rapid pan;
5. rapid zoom;
6. stationary refinement after motion;
7. drag selected node;
8. select highest-degree node;
9. cold tile cache;
10. warm tile cache;
11. label-heavy state;
12. filter or topology update.

## Report

At minimum:

```text
reference hardware + browser/runtime
viewport dimensions + DPR
p50/p95/p99 frame time
CPU main-thread time
GPU render time
GPU compute time
visible primitives
submitted primitives
upload bytes/frame
resident memory
cache hit ratio if streamed
picking latency
layout iteration time + layout-quality metric
```

Never publish “handles one million nodes” without the edge count, viewport, device,
interaction state, representation mode, and measured frame-time distribution.

---

# 20. Acceptance criteria

Project-specific targets should be explicit and versioned.

Example **default** acceptance policy for a 60 Hz desktop target:

```text
reference steady-state p95 frame <= 16.67 ms
reference steady-state p99 frame <= 25 ms
no synchronous GPU readback in pointer-move path
camera motion never waits for fine tile availability
selected semantic objects survive LOD transitions
LOD refinement is bounded per frame
layout compute has an explicit time/iteration budget
memory stays below configured device budget
```

These are engineering defaults, not claims from the literature.

For integrated/mobile-class hardware, set separate targets rather than pretending one
configuration is portable.

---

# 21. Implementation sequence

When building a new library or rescuing a slow one, prefer this order.

## Phase 0 — observability

Implement frame/GPU timers and counters before optimization.

## Phase 1 — flat GPU-resident baseline

- compact buffers;
- instanced nodes;
- indexed edge endpoints;
- stable pipelines/bindings;
- no per-edge CPU geometry rebuild;
- picking baseline.

Record the crossover point where this architecture stops satisfying the target.

## Phase 2 — packetization and visible-set control

- graphlets/edgelets;
- viewport rejection;
- GPU compaction if useful;
- hard output caps;
- temporal visible-set cache.

## Phase 3 — representation ladder and scheduler

- cluster hierarchy;
- screen-space/semantic error;
- coarse representations;
- cost/benefit allocator;
- hysteresis;
- motion-dependent quality.

## Phase 4 — edge-density strategies

Add only those required by the product task:

- density rendering;
- sampling;
- aggregation;
- bundling;
- matrix alternative.

## Phase 5 — out-of-core streaming

- tile pyramid;
- fixed resident cache;
- coarse fallback;
- prefetch/cancel;
- bounded uploads.

## Phase 6 — advanced layout

Only after rendering is understood:

- multilevel layout;
- GPU Barnes-Hut/quadtree;
- local refinement;
- asynchronous layout.

Do not optimize layout to 1 ms while edge rasterization costs 40 ms.

---

# 22. Frame algorithm template

Use this as conceptual pseudocode, not a mandatory implementation.

```text
function frame(view, interaction, dt):
    timings = consume_previous_gpu_timings()
    cost_model.update(timings)

    budgets = frame_budget_controller.next(view.motion, timings)

    frontier = lod_frontier.update_incrementally(
        previous_frontier,
        view,
        interaction.focus,
        max_work = budgets.lod_ms
    )

    reps = choose_minimum_valid_representations(frontier)

    upgrades = []
    for region in frontier:
        for next_rep in valid_upgrades(region):
            d_cost = cost_model.predict(region, next_rep, view)
            d_benefit = benefit_model.predict(region, next_rep, view, interaction)
            upgrades.push(region, next_rep, d_benefit / max(d_cost, epsilon))

    sort_descending(upgrades by value)

    predicted = cost(reps)
    for u in upgrades:
        if predicted + u.d_cost <= budgets.render_compute_ms:
            reps.apply(u)
            predicted += u.d_cost

    stream.request_missing_detail(reps, view, budgets.upload_bytes)

    gpu.classify_and_compact(reps)
    gpu.run_bounded_layout_if_enabled(budgets.layout_ms)
    gpu.render_aggregates()
    gpu.render_edges()
    gpu.render_nodes()
    gpu.render_labels()
    gpu.render_pick_ids_if_needed()

    preserve_semantic_interaction_state()
```

A more sophisticated scheduler may avoid a full sort by using buckets/heaps or by
updating only changed candidates.

---

# 23. Public library interface guidance

Separate graph semantics from rendering implementation.

Conceptual interfaces:

```ts
interface GraphStore {
  getNode(id: NodeId): NodeRecord | undefined;
  getEdge(id: EdgeId): EdgeRecord | undefined;
  queryNeighborhood(id: NodeId, options?: NeighborhoodQuery): Promise<Subgraph>;
}

interface GraphRenderer {
  setGraph(source: GraphSource): void;
  setView(view: ViewState): void;
  setSelection(selection: SelectionState): void;
  render(frame: FrameContext): void;
  getStats(): RenderStats;
}

interface LodPolicy {
  classify(
    region: RegionStats,
    view: ViewState,
    task: TaskContext,
  ): LodDecision;
  benefit(
    from: Representation,
    to: Representation,
    context: LodContext,
  ): number;
}

interface LayoutEngine {
  step(budget: LayoutBudget): LayoutStepResult;
  pin(ids: readonly NodeId[]): void;
  release(ids: readonly NodeId[]): void;
}

interface TileSource {
  request(
    keys: readonly TileKey[],
    signal: AbortSignal,
  ): Promise<readonly GraphTile[]>;
}

interface PerformancePolicy {
  targetFrameMs: number;
  maxUploadBytesPerFrame: number;
  maxVisibleNodes: number;
  maxVisibleEdges: number;
  maxVisibleLabels: number;
}
```

Do not expose WebGPU buffer offsets as the canonical application API.

---

# 24. Correctness review

Before calling an optimization complete, verify:

### Identity

- selection survives compaction;
- selection survives clustering/refinement;
- tile reload does not change semantic identity;
- duplicate/parallel edges remain distinguishable when required.

### Bounds

- packet/tile bounds include all represented primitives;
- culling never drops selected/focused content incorrectly;
- viewport/DPR transforms use consistent coordinate systems.

### Aggregation

- aggregate counts/weights conserve declared quantities;
- filtered elements are represented consistently;
- directionality is not silently lost;
- self-loops and multiedges have explicit behavior.

### Buffers

- chunk boundaries do not drop the final element;
- index width is sufficient;
- no out-of-bounds shader indexing;
- `NaN`/`Inf` positions are detected or quarantined;
- zero-sized buffers/graphs are handled.

### Temporal behavior

- physics does not depend unintentionally on FPS;
- LOD hysteresis prevents thrash;
- stale async tile results cannot overwrite newer camera state;
- layout and renderer agree on graph version.

---

# 25. Adversarial performance review

After the implementation appears correct, assume it is wrong and try to disprove the
claimed speedup.

For every optimization, answer:

```text
What exact resource should decrease?
What counter should show the decrease?
What new overhead is introduced?
At what graph/device size should the optimization cross over?
What workload should make it lose?
Can the same gain be achieved with a simpler change?
Does visual/semantic fidelity change?
```

Examples:

### Claim: packetization improves performance

Falsifier:

- flat instancing is faster at all target graph sizes after including compaction and
  packet metadata overhead.

### Claim: image-space aggregation scales with viewport rather than graph size

Falsifier:

- uniformly distributed nodes saturate the screen so seed/fragment work approaches
  the pixel-bound worst case and edge passes still scale with `E`.

### Claim: GPU layout is superior

Falsifier:

- integrated-GPU or small-graph benchmarks have worse total latency than CPU layout.

### Claim: hierarchy gives constant-ish frame time

Falsifier:

- a pathological high-degree region or selected supernode causes frontier expansion
  proportional to the total graph.

### Claim: bundling improves usability

Falsifier:

- task accuracy falls because bundled geometry implies false adjacency or hides
  individual paths.

---

# 26. Evidence discipline

When making architecture claims, distinguish:

- **Observation** — directly measured in this project or reported by a cited paper.
- **Assumption** — required condition not yet verified.
- **Inference** — mechanism inferred from observations.
- **Experiment** — proposed test that could falsify the inference.

For nontrivial optimization proposals, provide at least one falsifier.

Do not use literature-era absolute FPS numbers as promises for modern code. Hardware,
APIs, viewport sizes, graph distributions, shader complexity, and interaction semantics
differ materially.

---

# 27. Research-derived design principles

The following are the main transferable results encoded by this skill.

## Interactive LOD Rendering of Large Graphs — Zinsmaier et al. (2012)

Source: https://graphics.uni-konstanz.de/publikationen/Zinsmaier2012InteractiveLevelDetail/index.html

Transfer:

- screen-space node density can bound aggregate work by occupied pixels rather than
  directly by node count;
- node aggregation can guide edge aggregation;
- real-world clustered distributions behave differently from uniform random worst cases;
- image-space aggregation is fast but loses direct object provenance unless supported
  by a secondary mapping.

## GraphWaGu — Dyken et al. (2022)

Source: https://www.willusher.io/publications/graphwagu/

Transfer:

- WebGPU compute + storage buffers support a shared GPU-resident graph representation;
- edges should index current node positions rather than duplicating/re-uploading them;
- layout and rendering can share storage directly;
- very few instanced draw calls avoid per-entity CPU submission;
- GPU advantage is device/workload dependent, especially for compute layout.

## Interactive Graph Layout of a Million Nodes — Mi et al. (2016)

Source: https://doi.org/10.3390/informatics3040023

Transfer:

- multilevel graph structure can approximate long-range forces;
- avoiding repeated expensive spatial-structure construction can matter as much as
  arithmetic throughput;
- layout structures should be designed around GPU-resident access patterns.

## ZAME — Elmqvist et al. (2008)

Source: https://www.microsoft.com/en-us/research/publication/zame-interactive-large-scale-graph-visualization/

Transfer:

- use multiresolution pyramids;
- split large visual data into fixed-size tiles;
- maintain a bounded GPU cache;
- use coarse impostor data while high-resolution tiles load;
- prefetch according to interaction direction;
- memory hierarchy and IO are rendering concerns, not afterthoughts.

## ASK-GraphView — Abello, van Ham, Krishnan (2006)

Source: https://doi.org/10.1109/TVCG.2006.120

Transfer:

- hierarchical graph exploration separates total graph scale from currently expanded
  detail;
- interactivity and global detail are explicit tradeoffs;
- top-down cluster expansion is a graph-native form of semantic LOD.

## BubbleNet / Interactive Multi-resolution Large Graph Exploration — Lin et al. (2013)

Source: https://doi.org/10.1109/ICDMW.2013.124

Transfer:

- automatically generated graph decompositions can provide multiresolution exploration
  even without a pre-existing natural hierarchy;
- local/drill-down interaction can avoid holding or drawing full edge sets.

## CUBu — van der Zwan, Codreanu, Telea (2016)

Source: https://doi.org/10.1109/TVCG.2016.2515611

Transfer:

- edge-density work maps well to massively parallel GPU execution;
- gather formulations can substantially reduce write contention;
- projected edge sampling density should be treated as a quality/performance control;
- edge bundling is an LOD/clarity technique with semantic costs.

## FM³ — Hachul & Jünger

Source: https://doi.org/10.1007/978-3-540-31843-9_29

Transfer:

- multilevel force layouts reduce global interaction cost;
- hierarchical approximations matter more than exact pairwise forces at large scale;
- layout complexity should be treated separately from rendering complexity.

## Multi-Level Graph Layout on the GPU — Frishman & Tal (2007)

Source: https://doi.org/10.1109/TVCG.2007.70580

Transfer:

- mapping irregular topology into balanced GPU work is a primary design problem;
- GPU algorithms need data structures designed for parallelism and locality, not a
  literal port of pointer-oriented CPU algorithms.

## Graph Mapping — Jonker et al. (2017)

Source: https://doi.org/10.1177/1473871616661195

Transfer:

- massive graph visualization can be organized as hierarchical communities + recursive
  layout + tile summaries + map-like multiscale navigation;
- computation, memory, rendering, and semantic community quality are separate axes;
- tile pyramids enable browser clients to explore datasets larger than local memory.

## Atlas — Abello et al. (2019)

Source: https://doi.org/10.1145/3301275.3302275

Transfer:

- task-specific decomposition/layers can outperform a single universal node-link view;
- semantic representations should remain coordinated and inspectable;
- very large graphs benefit from local exploration in global context.

## Adaptive Display Algorithm — Funkhouser & Séquin (1993)

Source: https://doi.org/10.1145/166117.166149

Transfer:

- choose LOD predictively from estimated cost and benefit under a frame budget;
- screen coverage is a useful benefit component but semantic importance/focus matter;
- exploit frame-to-frame coherence;
- quality regulation should avoid reactive oscillation;
- omission is a legitimate representation when the alternative breaks the budget.

## View-Dependent Refinement of Progressive Meshes — Hoppe (1997)

Source: https://doi.org/10.1145/258734.258843

Transfer:

- maintain an incrementally changing active frontier;
- use screen-space error and view dependence;
- amortize refinement across frames;
- use hysteresis/morphing-like strategies to avoid popping;
- frame-rate regulation can control refinement aggressiveness.

## Geometry Clipmaps — Losasso & Hoppe (2004)

Source: https://doi.org/10.1145/1015706.1015799

Transfer:

- keep a bounded nested multiresolution working set centered on the view;
- prefer regular GPU-friendly storage even when it includes redundant samples;
- update only regions exposed by view movement;
- cap update bandwidth;
- degrade gracefully under rapid movement;
- coarse-to-fine streaming naturally supports remote data.

## Hierarchical Geometric Models — Clark (1976)

Source: https://doi.org/10.1145/360349.360354

Transfer:

- total database complexity can grow while visible complexity remains bounded;
- hierarchy plus frame coherence creates a graphical working set;
- this is the core scalability invariant for graph rendering as well.

## CHC++ — Mattausch, Bittner, Wimmer (2008)

Source: https://doi.org/10.1111/j.1467-8659.2008.01119.x

Transfer:

- exploit temporal/spatial coherence;
- batch tests/work rather than issuing fine-grained queries independently;
- predict visibility from previous state;
- for graph edges, transfer the scheduling principle, not necessarily Z-occlusion itself.

## Meshlet Generation Strategies — Jensen, Frisvad, Bærentzen (2023)

Source: https://jcgt.org/published/0012/02/01/

Transfer:

- insert an intermediate work granularity between whole scene and primitive;
- cluster data to improve culling, dispatch, and locality;
- packet construction strategy can materially affect rendering performance;
- graphlets/edgelets should therefore be benchmarked as data-layout choices, not treated
  as merely organizational abstractions.

## Hierarchical Z-Buffer Visibility — Greene, Kass, Miller (1993)

Source: https://doi.org/10.1145/166117.166147

Transfer:

- pair object-space hierarchy with image-space hierarchy for early termination;
- in 2D transparent graphs, substitute density/coverage hierarchy for literal Z rejection
  when occlusion is weak.

---

# 28. Default decision table

| Condition                            | First technique to try                                         |
| ------------------------------------ | -------------------------------------------------------------- |
| CPU submission dominates             | instancing, packetization, stable bind groups                  |
| CPU recomputes edge positions        | endpoint-index edges + GPU dereference                         |
| fragment/raster work dominates       | edge aggregation, sampling, density, filtering                 |
| full graph fits GPU but frame misses | hierarchical LOD + active frontier                             |
| graph exceeds GPU memory             | tile pyramid + resident cache + coarse fallback                |
| camera motion causes spikes          | bounded updates + motion-dependent refinement                  |
| LOD pops/thrashes                    | hysteresis + incremental frontier                              |
| layout dominates                     | multilevel/Barnes-Hut, lower layout Hz, GPU/CPU crossover test |
| integrated GPU is slow               | test CPU layout or simpler flat render path                    |
| picking stalls                       | async ID readback or CPU visible-set spatial index             |
| labels dominate                      | independent label LOD + cap                                    |
| high-degree selection explodes work  | aggregate/sample incident edges + progressive reveal           |
| hierarchy misleads analysis          | semantic hierarchy or alternate coordinated view               |
| dense overview conveys little        | matrix/density/community-map representation                    |

---

# 29. Final review checklist

Before finalizing a graph-rendering design or optimization, answer all of the following.

## Requirements

- What graph sizes and edge/node ratios are target workloads?
- What devices/runtime/browser are targets?
- What frame/interaction latency is required?
- Must topology change interactively?
- Must every individual edge remain inspectable?
- Is exact graph provenance required at every LOD?
- Can data be preprocessed/server-tiled?

## Mechanism

- What work scales with total graph size?
- What work scales with visible working-set size?
- What bounds the active working set?
- What degrades when the budget is exceeded?
- What remains semantically invariant across LODs?

## Alternatives

Compare at least:

1. flat GPU-resident rendering;
2. hierarchical LOD;
3. streamed/tiled LOD;

when more than one is plausible.

## Falsification

- What graph distribution breaks the preferred approach?
- What hardware makes it slower?
- What user task exposes semantic loss?
- What memory boundary invalidates the design?
- What simpler baseline could perform equally well?

## Verification

- Is the speedup visible in the expected profiler counter?
- Does p95/p99 improve, not only average FPS?
- Is interaction latency preserved?
- Is visual/semantic fidelity tested?
- Are selected and queried entities stable across LOD transitions?

If these questions cannot be answered, the optimization is not yet adequately specified.
