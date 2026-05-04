---
title: "03 — PyReason Propagation"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Tutorial 03 — PyReason propagation

The native evaluator from [tutorial 02](02-rules-and-derivations.md) is great when reasoning is a one-shot pull from the ledger. It's not the right tool when *beliefs propagate through a graph* — when the truth of a fact about one entity *depends on what's now true about its neighbours*, and that in turn depends on theirs, and so on, possibly over time. That's what PyReason is for.

This tutorial walks through a concrete propagation scenario: an outage in a service dependency graph. We have services that depend on each other; when one is down, services that depend on it become at-risk; when one of those is down, anything depending on *it* becomes at-risk; and so on. We'll build the graph, declare propagation rules, run PyReason, and accept the resulting facts back into our ledger.

Reach for this tutorial when:

- your data has a graph shape, with typed relationships you care about,
- a fact about one entity should *propagate* to its neighbours through some rule,
- you have time as part of the model (or could),
- intervals — *"this signal is true with confidence between 0.8 and 0.95"* — make sense for your problem.

Otherwise, the native evaluator is enough.

## Setup

PyReason ships as an optional dependency:

```
pip install factpy-kernel[pyreason]
```

The imports for this tutorial:

```python
from kernel.sdk import SDKStore, Entity, Identity, Field, Relationship
from kernel.sdk.compile import compile_schema_from_classes
from kernel.sdk.dsl.expr import LogicVar, Pred
from kernel.sdk.dsl.rule import Rule

from kernel.adapters.pyreason.session import PyReasonSession
from kernel.adapters.pyreason.runner import PyReasonRunConfig, run_pyreason
from kernel.adapters.pyreason.accept import accept_pyreason_session
```

The first block is the regular factpy DSL; the second block is the PyReason adapter surface. The session, runner, and accept calls are the three handles you'll touch.

## A graph-shaped schema

PyReason works on entities connected by typed relationships. A `Service`, with a few risk signals as fields, and a `DependsOn` relationship between services:

```python
class Service(Entity):
    service_id: str = Identity(primary_key=True)
    at_risk: str = Field(cardinality="single")           # boolean as string
    contingency_gap: str = Field(cardinality="single")

class DependsOn(Relationship):
    from_entity = "Service"
    to_entity = "Service"
    critical_path: str = Field(cardinality="single")     # boolean as string
```

A few notes:

- We use the `Relationship` base class here (not `Entity`) because PyReason treats edges as first-class — they're what propagation walks. For native-engine code, modelling relationships as plain entities with two `entity_ref` fields is fine; for PyReason, the typed `Relationship` form is what fits.
- Boolean signals are encoded as the strings `"true"` and `"false"` rather than Python `bool`. This is the convention PyReason expects in the adapter; conversion to typed booleans happens in the engine.
- We compile the schema explicitly because we need the `schema_ir` to construct a `PyReasonSession`:

```python
schema_ir = compile_schema_from_classes([Service, DependsOn])
```

## A session, with seeds

A `PyReasonSession` is where you build the graph state PyReason will run on. It has its own batch interface, similar to the SDK's:

```python
session = PyReasonSession(schema_ir)

with session.batch() as tx:
    # Three services
    db      = tx.entity(Service, service_id="db-primary")
    api     = tx.entity(Service, service_id="api-gateway")
    mobile  = tx.entity(Service, service_id="mobile-app")

    # The seed: db-primary is down with full confidence
    db.at_risk.set("true",
                   bound=[1.0, 1.0],
                   meta={"source": "incident_triage"})

    # The dependency edges, with their criticality also as confidence intervals
    tx.relationship(DependsOn,
                    from_entity=db, to_entity=api,
                    critical_path="true", bound=[1.0, 1.0],
                    meta={"source": "dependency_map"})
    tx.relationship(DependsOn,
                    from_entity=api, to_entity=mobile,
                    critical_path="true", bound=[1.0, 1.0],
                    meta={"source": "dependency_map"})
```

Two things are different from the SDK batch you saw in tutorial 01:

1. `bound=[lo, hi]` — every fact carries a confidence *interval*, not a single value. `[1.0, 1.0]` means "we're certain". `[0.8, 0.95]` would mean "we're between 80% and 95% confident". The interval form is core to PyReason; it lets the engine carry uncertainty through propagation properly.
2. `tx.relationship(...)` writes typed edges. Each edge can carry its own fields and confidence.

We've now described a small graph: three services, two edges, one seed. PyReason will propagate from the seed.

## Propagation rules

The rules look like SDK rules, but they encode *what propagates from where to where*:

```python
x = LogicVar("x")
y = LogicVar("y")

rules = [
    Rule(
        id="risk_propagation",
        version="1.0",
        select=[Pred("service:at_risk", x)],
        where=[
            Pred("service:at_risk", y),
            Pred("dependson:critical_path", y, x),
        ],
    ),
]
```

Read the rule: *if `y` is at risk, and there's a critical-path dependency from `y` to `x`, then `x` is at risk.* PyReason will apply this rule iteratively, propagating the at-risk signal from `db-primary` to `api-gateway`, then to `mobile-app`.

Notice that this looks structurally like the rules from tutorial 02 — same DSL, same atoms. The difference is in *how it runs*: PyReason iterates the rule across the graph until propagation stabilises (or a step limit is reached), rather than evaluating it once.

## Running PyReason

`run_pyreason` takes the session, the rules, and a config:

```python
result = run_pyreason(
    session,
    rule_defs=rules,
    config=PyReasonRunConfig(timesteps=4, atom_trace=True),
)
```

`timesteps=4` lets propagation walk up to four hops; for our two-edge chain, that's plenty. `atom_trace=True` records the per-step changes to the interpretation, which is what makes the timeline-flavoured audit story work later.

The `result` carries the engine's final interpretation, the elapsed time, and the per-timestep traces. You can inspect what changed:

```python
interp = result.interpretation.get_dict()
print(f"propagation took {result.elapsed_seconds:.1f}s")
for service_id, labels in interp.items():
    print(f"  {service_id}: {labels}")
```

If everything went well, all three services now carry `at_risk=true` — the seed at `db-primary`, the propagation to `api-gateway`, and the further propagation to `mobile-app`.

## Accepting results back

Inference is just inference — until something writes the conclusions back, the rest of your system doesn't see them. The adapter provides an explicit accept step that translates engine output into ledger writes:

```python
sdk = SDKStore.from_schema_classes([Service, DependsOn])
accept_result = accept_pyreason_session(sdk, result)

print("written assertions:", accept_result.assertion_count)
```

What this does, in shape, is the same as the candidate-and-accept flow from tutorial 02: every engine-derived fact becomes a candidate; accept writes it as a new assertion in the ledger; the engine's reasoning trace is preserved as the new fact's evidence. The audit story is consistent — *every fact in the ledger is either directly written or accepted from a derivation, with provenance back to the engine that produced it*.

You can verify the writes the usual way:

```python
mobile = sdk.get(Service, service_id="mobile-app")
print(mobile.at_risk)
# "true"  — propagated from db-primary through api-gateway
```

## The audit surface for timelines

PyReason runs produce *timeline-shaped* reasoning traces — at each step, certain atoms got new bounds because certain rules fired on certain neighbours. The audit story for that is richer than a flat evidence tree; it has its own surface in the audit module:

```
explain-timeline             # full per-step interpretation history
explain-timeline-summary     # condensed version
explain-timeline-narrative   # human-readable form
```

These are documented in [audit and provenance](../concepts/audit-and-provenance.md) and in (when written) the audit reference. The point for this tutorial: PyReason output, exported into an audit package, gives a reviewer the *why* of every propagated fact at a per-timestep level. *"`mobile-app` became at-risk at t=2 because `api-gateway` was at-risk at t=1 and there's a critical-path edge from `api-gateway` to `mobile-app`"* is the kind of explanation that falls out, not something you have to reconstruct.

## Recap

In this tutorial you've:

- declared a graph schema with `Entity` and `Relationship`,
- built a `PyReasonSession` with boolean seeds and confidence-bounded edges,
- written propagation rules in the same DSL you saw in tutorial 02,
- run PyReason with a step-bounded config and inspected the resulting interpretation,
- accepted the engine's output back into a regular factpy ledger with audit provenance preserved.

PyReason is the right tool when reasoning *propagates* through a typed graph. Like every adapter, it doesn't replace the SDK or the audit story; it plugs into them, takes a particular reasoning shape, and returns to the same candidate-and-accept rhythm the rest of factpy uses.

## Where to next

- [Tutorial 04 — ProbLog probabilistic](04-problog-probabilistic.md) for genuinely probabilistic logic over uncertain facts.
- [Tutorial 05 — Onboarding journey](05-onboarding-journey.md) for an end-to-end scenario weaving native rules, derivations, and adapters together.
- [Using adapters guide](../guides/using-adapters.md) for the reference angle on when to reach for which engine.