---
title: "Using Adapters"
weight: 8
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Using adapters

The native evaluator covers the rule shape most application code needs: typed bodies with joins, sub-rules, and negation as failure. For reasoning shapes that go beyond that — propagation through a graph of relationships, probabilistic inference over uncertain facts, large-scale Datalog over many millions of rules — factpy delegates to engine adapters. Three are shipped with the kernel: PyReason, ProbLog, and Souffle.

This guide is about *choosing* an adapter and the rough shape of how each plugs in. The full APIs live in each adapter's reference page; what's covered here is the decision and the integration pattern, not every option.

## Choosing an adapter by reasoning shape

Pick by the *shape of the reasoning*, not by preference. The native evaluator is the right answer until you have a concrete reason to leave it.

**PyReason** is for *graph-based annotated logic*. Beliefs are intervals (a lower and upper bound), they propagate along edges through time, and the natural data shape is *entities and relationships forming a graph that has a state, with inference re-running as the graph updates*. If you have entities connected by typed relationships, signals propagating through them with confidence bands, and a clock — that's PyReason.

**ProbLog** is for *probabilistic logic programming*. Facts have probabilities; rules combine them; the answers are probabilities over outcomes. If your rules need to reason about uncertain inputs in a principled probabilistic way — marginalising, computing joint probabilities, learning from data — that's ProbLog.

**Souffle** is for *fast Datalog at scale*. It compiles your rules into native code and evaluates them efficiently over very large fact bases. If your rule set has grown beyond what the native evaluator handles in reasonable time, and the rules themselves are well-formed Datalog (no probabilistic semantics, no graph propagation) — that's Souffle.

If none of those shapes describes your problem, you probably don't need an adapter. The native evaluator is uniform and good.

## ProbLog: probabilities on bodies

ProbLog has the closest-to-uniform integration. Import the adapter to register the engine, then evaluate a derivation with `mode="problog"`:

```python
import kernel.adapters.problog  # registers the engine evaluator

candidates = sdk.evaluate(some_derivation, mode="problog")
```

For derivations that need engine-specific configuration — branch probabilities, inference parameters — attach a `ProbLogRuleExt` to the rule:

```python
from kernel.adapters.problog.rule_ext import ProbLogRuleExt

with vars("r", "name") as (r, name):
    expertise_drv = Derivation(
        id="drv.expertise",
        version="v1",
        where=[Pred("researcher:name", r, name)],
        head=Researcher.expertise(researcher_id=r, expertise=name),
        mode="problog",
        engine_ext=ProbLogRuleExt(branch_probabilities=(0.85,)),
    )
```

Candidates produced by the ProbLog evaluator carry probability annotations forward. Acceptance writes them as conventional confidence metadata on the resulting ledger assertions, and they show up in the audit story like any other derived fact — with the difference that the evidence graph reflects the engine's reasoning trace rather than a flat tree.

The `Body(..., confidence=...)` shape covered in [running derivations](running-derivations.md) is the lightweight handoff: a derivation with weighted bodies in the native evaluator becomes a derivation with probabilistic bodies in ProbLog with no DSL change. When the native version stops being expressive enough — you need joint distributions, not just per-body weights — the same `Derivation` object runs under ProbLog by switching `mode`.

## PyReason: graph propagation as a session

PyReason's integration is shaped differently because the engine itself is. Inference doesn't happen against a static ledger as a one-shot call; it propagates through a graph that changes over time. The adapter exposes that as a *session* — a stateful object you build up the graph in, then run inference on:

```python
from kernel.adapters.pyreason.session import PyReasonSession
from kernel.adapters.pyreason.runner import PyReasonRunConfig, run_pyreason
from kernel.adapters.pyreason.accept import accept_pyreason_session

session = PyReasonSession(schema_ir)

with session.batch() as tx:
    a = tx.entity(Vendor, vendor_id="A")
    b = tx.entity(Vendor, vendor_id="B")
    a.at_risk_signal.set("true", bound=[1.0, 1.0],
        meta={"source": "incident_triage"})
    tx.relationship(VendorDependency, from_entity=a, to_entity=b,
        critical_path="true", bound=[1.0, 1.0])

result = run_pyreason(session, PyReasonRunConfig(...))
accept_pyreason_session(sdk, result, ...)
```

The pattern: build the typed graph in a session, run propagation, accept the resulting facts back into the main store. The adapter takes care of translating between factpy's schema and PyReason's annotated-graph representation; you write entity declarations as you would normally, with the addition of `bound=[lo, hi]` for confidence intervals and explicit `Relationship` declarations.

Use PyReason when the reasoning is propagation over a graph state. The session-based API is what makes that shape ergonomic; trying to express the same thing as a flat derivation under the native engine fights the model.

## Souffle: package-based execution

Souffle is a compiled Datalog engine. The integration reflects that: factpy exports a *package* describing the rules, the schema, and the input facts; Souffle compiles and runs against it; results come back as new facts:

```python
from kernel.adapters.souffle.package import ExportOptions

# 1. Export an inference package
out = Path("./packages/run-1")
sdk.export_package(out, ExportOptions(package_kind="inference"))

# 2. Run it under Souffle
result = sdk.run_package(out, entrypoints=["my_rule"], engine="souffle")
```

`package_kind="inference"` is the leaner package — schema, rules, input facts, no audit overhead. `entrypoints` names the rules whose outputs you want. `run_package` invokes the Souffle binary against the package and returns the derived facts.

Reach for Souffle when rule evaluation under the native engine is the bottleneck. The cost is: a package round-trip per run, a binary dependency on `souffle`, and a less interactive feedback loop than running rules in-process. The benefit is throughput on rule sets that the native evaluator would not handle in acceptable time.

The same `Rule` and `Derivation` declarations work — the adapter compiles them into Souffle's surface language. You don't rewrite rules; you change *how* they run.

## What stays uniform across adapters

Three things are the same regardless of which adapter (or none) you use:

- **Schema and DSL.** The same entity declarations, the same `Rule` and `Derivation` objects. An adapter changes the evaluator, not the language.
- **Audit story.** Every adapter produces evidence — as a tree under the native engine, as a richer `EvidenceGraph` under engines that don't reduce to trees. Audit packages capture both shapes uniformly.
- **The candidate-and-accept rhythm** (where applicable). For ProbLog and the parts of Souffle that produce candidates, evaluate-then-accept works as covered in [running derivations](running-derivations.md). PyReason's session has its own commit step (`accept_pyreason_session`) that plays the same role for that engine's output.

What's *not* uniform: the surface for engine-specific configuration (branch probabilities, propagation steps, Souffle compile flags), the shape of the evidence (tree vs. graph vs. timeline), and the operational dependencies (PyReason and ProbLog as Python packages; Souffle as a compiled binary). The adapters' reference pages cover these in detail.

## Operational notes

PyReason and ProbLog ship as optional Python dependencies. They are not installed by default; install them only when you need them (`pip install factpy-kernel[pyreason]`, `pip install factpy-kernel[problog]` — confirm exact extras in the project's install docs). Importing the adapter modules without the underlying engine installed raises a clear error.

Souffle requires the `souffle` binary on `PATH`. The adapter doesn't ship the compiler.

When developing rules locally, work against the native evaluator. It has no extra dependencies, fast iteration, and the smallest blast radius for mistakes. Move to an adapter when the *reasoning shape itself* needs it — not as a default.

## Where to next

- [Running derivations](running-derivations.md) for the candidate-and-accept lifecycle the ProbLog and Souffle adapters share with the native engine.
- [Auditing a run](auditing-a-run.md) for how adapter-produced evidence (graphs, timelines) shows up in audit packages.
- The adapter-specific reference pages (when written) cover full configuration, engine extensions, and the data conversions each performs.