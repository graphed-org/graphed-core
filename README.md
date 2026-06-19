# graphed-core

Rust+PyO3 **thread-safe interned graph IR, optimizer, and plan serialization** for `graphed` — the
spine every other package builds on. The graph lives in Rust, not Python. This package **MUST NOT
import awkward**. Part of the [`graphed-org`](https://github.com/graphed-org) project; see
[`graphed-project`](https://github.com/graphed-org/graphed-project-mvp) for the root guidance and the
authoritative plan.

## The interned IR (M1)

A `GraphStore` interns nodes by structural hash: structurally identical nodes share one `NodeId`, so
`node_count()` equals the number of distinct structural keys (CSE falls out of hash-consing). Nodes
are `Source | Op | Reduction | External | Stage`; `External` carries a `PayloadDescriptor` (kind,
content hash, framework+version, I/O schema, preprocessing ref) that participates in the hash. Float
params use a deterministic total order (NaN interns to itself; `0.0` and `-0.0` are distinct). The
store is `Send + Sync` (one documented `Mutex`, `loom`-modeled) and safe to build from many threads
under the GIL and free-threaded 3.14t. `to_dot()` is byte-stable.

```python
import graphed_core as gc
s = gc.GraphStore()
src = s.add_source("events", {"uri": "f.root", "tree": "Events"})
pt = s.add_op("pt", [src])
assert s.add_op("pt", [src]) == pt        # interned
s.mark_output(s.add_reduction("sum", [pt]))
```

## The optimizer (M4)

Reduction runs **equality saturation over an e-graph** behind a `RewriteEngine` trait, so no `egg`
types leak past the boundary (the Phase-2 `egglog` swap is one trait impl). Only **sound** rewrites
are used; extraction is an O(N) cost-quotient, not egg's O(depth·N) recursive extractor, so deep
systematics chains do not blow up. **DCE** (reachability from outputs) and **CSE** (hash-consing) are
plain passes outside the engine; **stage fusion** groups maximal op-runs between boundaries into
`Stage` nodes. Reduction is incremental and byte-stable, and a CI benchmark **fails on super-linear
reduction time** across {1k,2k,4k,8k} — the central guard against the O(N²) behavior that sank
dask-awkward.

## Plan serialization (M8)

`GraphStore.serialize()/deserialize()` is a versioned (`GIR1`), byte-identical binary codec — the
**canonical durable representation** (never cloudpickle). `DurablePlan` (pure Python, `plan.py`) wraps
the serialized IR with executor metadata (partitions, read columns, reduction/stop/locality/resource
specs); it is versioned, byte-identical, and content-addressed (`task_id`). `OpSpec` resolves
callables by import ref, embedding by value (cloudpickle, flagged `opaque=True`) only for genuinely
opaque callables.

## The execution contract + observability seams (`graphed_core.execution`)

This package also owns the *pure-Python, data-only* execution contract every executor implements:
`Plan`, `Task`, `Partition`, `StopCondition`, `Executor`, plus the dependency-free `SequentialRunner`
baseline. It imports no awkward/numpy/web — a stable, minimal seam. It hosts two cross-executor seams,
both transport-agnostic and passive (a raising hook never changes a result):

- **Live observability (M37):** `TaskEvent` / `TaskPhase` and the `Monitor` / `WorkerProfiler`
  protocols — an executor *emits* events a dashboard can watch; `graphed-debug` supplies the
  websocket/Perspective rendering.
- **Inter-worker comms (M38):** the `WorkerTransport` seam for peer tree-reduction and work-stealing
  (backends in `graphed-exec-local`).

The vocabulary lives here because it is shared by *every* executor — a shared primitive belongs at the
layer it serves.

## Develop

```bash
pip install -e ".[dev,docs]"
maturin develop                  # build the extension into the venv
uvx prek run --all-files         # ruff + mypy (python) and, in the rust CI job, cargo fmt + clippy
pytest tests/frozen              # M1 IR + M4 optimizer + M8 plan suites
cargo test                       # pure-Rust IR + locking stress + loom model
```

Guardrails: no awkward; the graph + its optimization live in Rust, not Python; any `unsafe` needs a
line-by-line `// SAFETY:` justification (there is none).
Status: see `.graphed/state.json` and `CLAUDE.md`.
