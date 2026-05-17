---
title: "Quickstart"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---


## Install

```bash
pip install factgraph
```

## Quickstart

Define a typed schema, write facts, and read the current snapshot:

```python
from factgraph.sdk import Entity, FactGraph, Field, Identity


class User(Entity):
    user_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    role_seed: str = Field(cardinality="single")
    roles: str = Field(cardinality="multi")


fg = FactGraph.create(schema_classes=[User])

alice = fg.read.ref(User, user_id="u-1")

name_id = fg.write.set(User.name, alice, "Alice")
fg.write.set(User.role_seed, alice, "engineer")

snap = fg.read.get(User, user_id="u-1")

assert snap is not None
assert snap.name == "Alice"
assert snap.role_seed == "engineer"
```

A write appends an assertion and returns an `asrt_id`. A read returns a snapshot resolved from the active assertions.

## Assertions And Views

Assertions are the durable records behind facts. They make precise lookup, review, retraction, and audit possible.

Graph-level assertion access lives under `fg.assertions`:

```python
record = fg.assertions.by_id(name_id)

assert record is not None
assert record.value == "Alice"
```

Snapshot-level assertion access lives under `snap.assertions`:

```python
name_record = snap.assertions.field("name").active().by_id(name_id).one()

assert name_record.asrt_id == name_id
```

Views are named selections of assertion ids:

```python
review_view = fg.views.create(
    "name_review",
    asrt_ids=[name_id],
)

records = fg.assertions.by_ids(review_view.asrt_ids)

assert records.one().value == "Alice"
```

A view is an assertion-level object:

```text
assertion ids -> named view -> selected assertion records
```

Views are useful for review sets, audit selections, and evidence workflows. They do not change the active fact projection and are not passed as `view=` parameters to `fg.eval.evaluate(...)`.

## Reasoning

`factgraph` includes a small logic DSL for describing patterns over facts.

A `Rule` reads matching facts. An `Inference` proposes new facts. Evaluation is read-only; it produces candidates. Accepting a candidate writes derived facts into the ledger.

```python
from factgraph.sdk import Branch, Inference, Pred, vars

with vars("u", "role") as (u, role):
    roles_from_seed = Inference(
        id="inf.roles_from_seed",
        version="v1",
        where=[
            Branch(
                [Pred("user:role_seed", u, role)],
                id="seed_path",
            )
        ],
        target="user:roles",
        head_vars=[u, role],
    )

candidates = fg.eval.evaluate(roles_from_seed)

assert len(candidates) == 1
assert tuple(fg.read.get(User, user_id="u-1").roles) == ()

fg.eval.accept(candidates[0])

assert tuple(fg.read.get(User, user_id="u-1").roles) == ("engineer",)
```

The lifecycle is explicit:

```text
Inference -> evaluate -> CandidateSet -> accept -> ledger assertion
```

This separation allows applications to inspect, approve, compare, or reject derived facts before committing them.

## What You Can Build With It

`factgraph` is useful when facts and reasoning need to remain inspectable:

- audit-friendly data models
- rule-based enrichment pipelines
- explainable inference workflows
- reviewable derived facts
- counterfactual checks over proposed fact or rule changes
- provenance-aware application state

## API Map

| Surface | Purpose |
|---|---|
| `fg.schema` | Add schema elements |
| `fg.read` | Create entity refs, read snapshots, find entities |
| `fg.write` | Set, add, and retract facts |
| `fg.assertions` | Inspect graph-level assertion records |
| `snap.assertions` | Inspect assertion records for one snapshot |
| `fg.views` | Create named assertion-id selections |
| `fg.rules` | Save, load, and inspect rules |
| `fg.inferences` | Save and load inference definitions |
| `fg.eval` | Run rules, evaluate inferences, accept candidates |
| `fg.what_if` | Check, diagnose, and explore counterfactuals |
| `fg.audit` | Explain persisted facts and compare proof frames |

## Persistence

Graphs can be kept in memory or backed by a workspace path.

```python
fg = FactGraph.create(schema_classes=[User], path="workspace")
fg.save()

restored = FactGraph.load("workspace", schema_classes=[User])
```

A workspace stores the ledger, schema metadata, registry, and manifest.

## Development

```bash
pixi run test
pixi run build
```

## License

Licensed under the Apache License, Version 2.0.

Copyright 2026 Symbolic Intelligence GbR.
