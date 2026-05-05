---
title: "Schnellstart"
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

# Quickstart

This page works through the smallest factpy program that exercises each of the kernel's central abstractions: a typed schema, an append-only ledger of writes, a logical rule evaluated against those writes, and the recovery of a fact's provenance from the resulting state. Everything below uses only the SDK's public surface, and nothing has been simplified away for presentation.

## Install

factpy requires Python 3.10 or newer.

```bash
pip install factpy-kernel
```

The kernel itself has a minimal dependency set, and the adapter engines for PyReason, ProbLog, and Souffle are optional extras that are not needed for what follows.

## A schema

We model a single entity, `Person`, whose primary identity is a string id, whose name is a single-valued field, and whose tags are a multi-valued field.

```python
from kernel.sdk import Entity, Identity, Field

class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")
```

The identity field marked `primary_key=True` is what locates an instance of `Person` uniquely in the ledger. The cardinality declared on each non-identity field determines how multiple assertions to that field reduce when the entity is projected: a `"single"`-cardinality field keeps the latest active assertion, while a `"multi"`-cardinality field keeps the union of active values. The python type annotations map to the kernel's typed value domains, here `string`. The fuller picture of how schemas are constructed and what guarantees they offer is laid out in [entities and fields](concepts/entities-and-fields.md).

## A store and a few writes

We open an in-memory store from the schema and write a small set of facts about three people.

```python
from kernel.sdk import SDKStore

sdk = SDKStore.from_schema_classes([Person])

alice = sdk.ref(Person, person_id="p-1")
sdk.set(Person.name, alice, "Alice", meta={"source": "import"})
sdk.add(Person.tag, alice, "vip", meta={"source": "policy"})

bob = sdk.ref(Person, person_id="p-2")
sdk.set(Person.name, bob, "Bob", meta={"source": "import"})
sdk.add(Person.tag, bob, "vip", meta={"source": "policy"})
sdk.add(Person.tag, bob, "blocked", meta={"source": "compliance"})

carol = sdk.ref(Person, person_id="p-3")
sdk.set(Person.name, carol, "Carol", meta={"source": "import"})
sdk.add(Person.tag, carol, "member", meta={"source": "policy"})
```

`SDKStore.from_schema_classes` opens a store backed, in this case, by an in-memory ledger; passing a `ledger_path` instead would persist every entry to disk and let the same data be reopened in a later process. `sdk.ref` returns a managed reference to the identity of an entity, which the SDK requires for subsequent writes against that entity. The two write primitives of the direct API are `sdk.set`, which writes an assertion to a single-cardinality field, and `sdk.add`, which writes an assertion to a multi-cardinality field; neither modifies an earlier entry. Each call appends a new ledger record carrying the predicate, the subject, the value, the time, and the metadata supplied at the call site.

## Reading

To read the current state of an entity's fields, ask for a snapshot.

```python
bob_snap = sdk.get(Person, person_id="p-2")
print(bob_snap.name)   # "Bob"
print(bob_snap.tag)    # {"vip", "blocked"}
```

A snapshot is a read-only projection of the ledger for one entity at the active view: single-cardinality fields appear as their latest active value, multi-cardinality fields as the set of all active values. The snapshot is recomputed from ledger entries on demand and is never stored as canonical state.

## A rule with negation

The rule below asks for the name of every person who carries the `vip` tag and does not carry the `blocked` tag.

```python
from kernel.sdk import Rule, Pred, Not, vars

with vars("p", "n") as (p, n):
    vip_ok = Rule(
        id="q.vip_ok",
        version="1.0.0",
        select=[n],
        where=[
            Person(p),
            Pred("person:tag", p, "vip"),
            Not([Pred("person:tag", p, "blocked")]),
            p.name == n,
        ],
    )

print(sdk.run(vip_ok, row_format="dict"))
# [{"n": "Alice"}]
```

The body of the rule introduces two logic variables, `p` ranging over `Person` instances and `n` over names, and then constrains them through three clauses: the first requires that `p` carry the `vip` tag, the second requires that `p` not carry the `blocked` tag, and the third binds `p`'s name to `n`. Bob is excluded by the second clause, Carol never satisfies the first, and Alice is the only match. The atom `Pred("person:tag", p, "vip")` and the typed shorthand `p.tag == "vip"` express the same condition; the predicate form is the more general of the two and is preferable wherever the predicate name is itself something the call site wants to read.

The semantics of `Not` is *negation as failure*: the clause succeeds whenever the ledger does not currently support its body, and fails otherwise. This is not the same as an explicit assertion of negation, which has no representation in the ledger — only the absence of an assertion of the corresponding positive claim. The practical consequences of this distinction, particularly for systems that ingest data over time and from multiple sources, are taken up in [writing rules](guides/writing-rules.md).

## Recovering provenance

In addition to the reduced field values, every snapshot exposes, for each field, the assertion history that the projection was computed from. Reading that history is the means by which one recovers what was claimed about an entity, when, and by whom.

```python
alice_snap = sdk.get(Person, person_id="p-1")

for asrt in alice_snap.assertions.tag.history:
    print(asrt.value, asrt.meta.get("source"), asrt.retracted)
# vip   policy   False
```

Each entry in the history carries the asserted value, the metadata as written at the time, the timestamp the kernel stamped on it, and a flag indicating whether a later retraction has superseded it. In Alice's case, the `tag` history contains a single entry, written by the `policy` source, still active. Had the same fact been asserted twice — by a policy import and by a downstream classifier, for instance — both entries would be present, each retaining the metadata of its origin, and the snapshot's reduction over them would be a convenience layered on top of the ledger rather than a replacement for it.

Every fact in the ledger remains traceable to the event that produced it, and every read can recover that origin. This is the property on which the rest of the system rests. For derivations, the same mechanism additionally records the rule and version that produced a fact, and `sdk.export_package` bundles the complete record of a run into a directory that can be reviewed, archived, or shared independently of any running store.

## Where to next

[Concepts → Overview](concepts/overview.md) lays out the model these moves rest on. [Tutorial 01](tutorials/01-sdk-basics.md) is a longer walkthrough that adds persistence, retractions, and composite identities. The [guides](guides/_index.md) are organised around specific tasks, and the [reference](reference/_index.md) covers the full API surface.