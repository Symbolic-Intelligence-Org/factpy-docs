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

The native evaluator covers the rule shape that most application code requires: typed bodies with joins, sub-rules, and negation as failure. For three reasoning shapes that the native evaluator does not address — propagation through a graph of typed relationships, probabilistic inference over uncertain facts, and Datalog evaluation at scale — the kernel delegates to adapter engines. Three are shipped: PyReason, ProbLog, and Souffle. The present guide develops the criteria by which an adapter is chosen and the integration pattern of each, with the configuration details and full APIs left to the adapter reference pages.

## Choosing an adapter

The choice is determined by the shape of the reasoning, not by preference. The native evaluator is the appropriate default until a concrete reason to leave it appears, since it has no further operational dependencies, the fastest iteration cycle, and the smallest blast radius for modelling mistakes.

PyReason is appropriate where the reasoning has the shape of *graph-based annotated logic*. Beliefs are intervals with a lower and an upper bound, they propagate along typed edges through discrete time steps, and the natural data shape is a graph of entities and relationships whose state evolves as inference re-runs. The adapter is the appropriate choice wherever entities are connected by typed relationships, where signals propagate through them carrying confidence bands, and where time is part of the model.

ProbLog is appropriate where the reasoning is *probabilistic logic programming* in the strict sense. Facts have probabilities, rules combine them, and the answers are probabilities over outcomes. The adapter is the appropriate choice wherever rules need to reason about uncertain inputs in a principled probabilistic way, including marginalising over alternatives, computing joint probabilities, and learning probabilities from labelled data. It is *not* the appropriate choice for systems whose confidences are heuristic labels rather than calibrated probabilities; in that case the native evaluator with weighted bodies is sufficient and considerably cheaper.

Souffle is appropriate where the reasoning is plain Datalog and the rule set has grown beyond what the native evaluator handles in acceptable time. The engine compiles rules into native code and evaluates them efficiently over very large fact bases. The adapter is the appropriate choice wherever throughput on a large fact base has become the bottleneck and the rules themselves are well-formed Datalog without probabilistic semantics or graph propagation.

If none of these shapes describes the problem at hand, no adapter is needed. The native evaluator is uniform and sufficient for the substantial majority of factpy projects.

## ProbLog: probabilities on bodies

ProbLog has the closest-to-uniform integration with the SDK. Importing the adapter package registers the engine with the dispatch in `sdk.evaluate`, after which a derivation can be run under ProbLog by passing `mode="problog"`.

```python
import kernel.adapters.problog  # registers the engine evaluator

candidates = sdk.evaluate(some_derivation, mode="problog")
```

For derivations that need engine-specific configuration — branch probabilities, inference parameters — a `ProbLogRuleExt` is attached to the derivation through its `engine_ext` parameter.

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

Candidates produced by the ProbLog evaluator carry probability annotations forward into acceptance, which writes them as conventional `confidence` metadata on the resulting ledger assertions. The accepted facts appear in the audit story like any other derived fact, with the difference that the evidence record reflects the engine's reasoning trace rather than a flat tree.

The `Body(..., confidence=...)` shape developed in [running derivations](/docs/guides/running-derivations) is the lightweight handoff between the two evaluators: a derivation with weighted bodies under the native evaluator becomes a derivation with probabilistic bodies under ProbLog without any change to the DSL. When weighted-body confidences cease to be expressive enough — when joint distributions and marginalisation become necessary — the same `Derivation` object runs under ProbLog by switching the `mode` argument.

## PyReason: graph propagation as a session

PyReason's integration is shaped differently because the engine itself is. Inference does not run as a one-shot evaluation against a static ledger; it propagates through a graph whose state changes over discrete time steps. The adapter exposes that shape as a *session* — a stateful object in which the typed graph is built up before propagation runs against it.

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

The integration pattern is to build the typed graph in a session, run the propagation, and commit the resulting facts back into the main store. The adapter handles translation between factpy's schema and PyReason's annotated-graph representation; entity declarations and relationships are written as they would be in the SDK, with the addition of `bound=[lo, hi]` for confidence intervals on each fact and explicit `Relationship` declarations for the typed edges along which beliefs propagate.

PyReason is the appropriate choice wherever the reasoning *is* propagation through a graph state. The session-based API is what makes that shape ergonomic; expressing the same propagation as a flat derivation under the native evaluator is awkward at best and rapidly becomes unworkable as the depth of propagation grows.

## Souffle: package-based execution

Souffle is a compiled Datalog engine, and the integration reflects that. The kernel exports a *package* describing the rules, the schema, and the input facts; the Souffle binary compiles and runs against the package; the results return as new facts.

```python
from kernel.adapters.souffle.package import ExportOptions

# Export an inference package
out = Path("./packages/run-1")
sdk.export_package(out, ExportOptions(package_kind="inference"))

# Run it under Souffle
result = sdk.run_package(out, entrypoints=["my_rule"], engine="souffle")
```

The `package_kind="inference"` setting produces the leaner package suitable for re-execution rather than for review — schema, rules, and input facts, without the audit-side records that an audit package additionally carries. The `entrypoints` argument names the rules whose outputs are wanted from the run. `run_package` invokes the Souffle binary against the exported package and returns the derived facts as a result.

Souffle is the appropriate choice when rule evaluation under the native engine has become the bottleneck. The cost of choosing it is a package round-trip per run, a binary dependency on `souffle` that must be installed on the host, and a less interactive feedback loop than running rules in-process. The benefit is throughput on rule sets and fact bases that the native evaluator would not process in acceptable time. The same `Rule` and `Derivation` declarations are used; the adapter compiles them into Souffle's surface language without further intervention from the rule author.

## What stays uniform across adapters

Three properties hold uniformly across the native evaluator and all three adapters. The schema and the DSL are unchanged: the same `Entity` declarations, the same `Rule` and `Derivation` objects. An adapter changes the evaluator that consumes them, not the language in which they are written. The audit story is unchanged in its general shape: every evaluator produces evidence, in the form of a tree under the native engine and as a richer `EvidenceGraph` under engines whose reasoning does not reduce to a tree, and audit packages capture both shapes uniformly. And the candidate-and-acceptance lifecycle is unchanged where applicable: ProbLog and the candidate-producing portions of Souffle plug into `sdk.accept` and `sdk.accept_many` exactly as the native evaluator does, while PyReason's session-based output flows through `accept_pyreason_session`, which plays the same role for that engine's results.

What is *not* uniform across adapters is the surface for engine-specific configuration — branch probabilities for ProbLog, propagation step counts and atom-trace flags for PyReason, dialect versions and policy modes for Souffle — and the shape of the evidence each engine produces, which is a tree, graph, or timeline depending on the engine's reasoning model. Operational dependencies likewise differ: PyReason and ProbLog ship as optional Python packages, while Souffle requires a separately installed compiled binary. The adapter reference pages cover each of these surfaces in detail.

## Operational notes

PyReason and ProbLog are optional Python dependencies and are not installed with the kernel by default. They are installed through factpy-kernel's optional extras (`pip install factpy-kernel[pyreason]`, `pip install factpy-kernel[problog]`); the exact extras names should be confirmed against the project's install documentation, since these may evolve. Importing the adapter modules without the underlying engine present raises a clear error rather than failing later in the run.

Souffle requires the `souffle` binary on the host's `PATH`. The adapter does not ship the compiler, and a missing binary surfaces as a preflight warning before it surfaces as a runtime error.

Local rule development is best done against the native evaluator. The native evaluator has no further dependencies, the fastest iteration cycle, and the smallest blast radius for modelling mistakes. Adapters are the appropriate choice when the reasoning shape itself requires them, not as a default for ordinary work.

## Where to next

[Running derivations](/docs/guides/running-derivations) covers the candidate-and-acceptance lifecycle that ProbLog and Souffle share with the native evaluator. [Auditing a run](/docs/guides/auditing-a-run) covers how adapter-produced evidence — graphs and timelines — appears in audit packages. The adapter-specific reference pages cover full configuration, engine extensions, and the data conversions each adapter performs.