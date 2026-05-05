---
title: "01 — SDK Basics"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Tutorial 01 — SDK basics

This tutorial walks from an empty project to a working factpy ledger you can read, write, correct, and persist. By the end you'll have written a small schema, populated it, queried it, fixed a bad assertion, and saved the result to disk. We don't assume any prior factpy code — only that you've skimmed [the overview](../concepts/overview.md) and roughly know what a fact-based store is.

The example domain is small on purpose: a registry of people and the countries they live in. The same patterns scale up; starting small makes the moves easier to see.

## Setup

Install the kernel:

```
pip install factpy-kernel
```

Then in a fresh python file:

```python
from kernel.sdk import (
    SDKStore, Entity, Identity, Field,
    schema_preflight_from_classes,
)
```

That's the whole import surface for this tutorial. Other entry points (`Rule`, `Derivation`, `Pred`) come in the next tutorial.

## A minimal schema

Three entities. A `Country` with a natural key. A `Person` with a composite identity (id plus locale, so the same person can have a localised view per language). And a `LivesIn` relationship as its own entity, with a surrogate key:

```python
class Country(Entity):
    iso_code: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")

class Person(Entity):
    person_id: str = Identity(primary_key=True)
    locale: str = Identity()
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")

class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
```

A few details worth noticing now and revisiting later:

- `cardinality="single"` for `name` means the snapshot has at most one current name. A new `set` doesn't overwrite history; it just changes what reduces into the snapshot.
- `cardinality="multi"` for `tag` means a person can carry many tags simultaneously. Multiple `add`s accumulate.
- `LivesIn` has `default_factory="uuid4"` because there's no meaningful natural key for "the relationship between this person and that country" — we let the SDK assign one.

If any of this is mysterious, the [defining-a-schema guide](../guides/defining-a-schema.md) covers the choices in depth.

## Validating the schema

Before opening a store, run `schema_preflight_from_classes`:

```python
classes = [Country, Person, LivesIn]
report = schema_preflight_from_classes(classes)
print("preflight:", report)
```

This catches issues — missing primary keys, unresolvable cross-entity references, bad type domains — without ever touching a ledger. You'd typically run this in your test suite. If it returns clean, the schema will open as a store.

## Opening an in-memory store

For this tutorial, an in-memory ledger is fine. We'll persist to disk at the end:

```python
sdk = SDKStore.from_schema_classes(classes)
print(sdk)
# SDKStore(entities=3, schema='...digest...')
```

The store is empty. Everything from here is about putting facts in and getting them back.

## Writing a first batch

The batch API groups multiple writes into one atomic unit. Open a transaction with optional metadata that flows onto every assertion the batch produces:

```python
tx = sdk.batch(meta={"source": "tutorial", "trace_id": "tut-01"})

# Two countries
de = tx.entity(Country, iso_code="DE")
de.name.set("Germany")

fr = tx.entity(Country, iso_code="FR")
fr.name.set("France")

# A person with full identity, in one step
alice = tx.entity(Person, person_id="p-001", locale="en")
alice.name.set("Alice")
alice.tag.add("vip")
alice.tag.add("early-adopter")

# A relationship
rel = tx.entity(LivesIn)
rel.person.set(alice)
rel.country.set(de)
rel.since.set(2019)
```

`tx.entity(Cls, **identity)` returns a *handle* on the entity. Method calls on the handle (`alice.name.set(...)`) queue writes on the batch; nothing reaches the ledger yet. You can preview what's queued:

```python
plan = tx.preview()
print("planned ops:", len(plan.ops))
```

And commit:

```python
result = tx.commit()
print("written assertions:", len(result.apply_result.assertion_ids))
```

Commit is atomic. Either every queued write makes it in, or — if the kernel rejects something at commit time — none of them are visible.

## Reading what we just wrote

Two ways to read. For one entity by identity, use `sdk.get`:

```python
snap = sdk.get(Person, person_id="p-001", locale="en")

print("ref:", snap.ref)
print("name:", snap.name)
print("tags:", snap.tag)
```

`snap` is a read-only snapshot — the projection of the ledger for this entity at the active view. Single-cardinality fields read as values, multi-cardinality fields read as sets. If you tried to assign to `snap.name`, you'd get a `FrozenSnapshotError`; snapshots are immutable.

For *which entities match a filter*, use `sdk.find`:

```python
vips = sdk.find(Person, tag="vip")
for row in vips:
    print(row.ref, row.name)
```

`find` returns a list of rows — each row is a thin entity-shaped object you can read fields off. For richer queries (joins, negation, sub-rules), you'd write a `Rule` and use `sdk.run` — but that's the next tutorial.

## Fixing a wrong fact

Suppose Alice's name was actually entered with a typo. We thought it was "Alice", but it should be "Alicia". This is the simplest correction — write a new fact:

```python
ref = sdk.ref(Person, person_id="p-001", locale="en")
sdk.set(Person.name, ref, "Alicia",
        meta={"source": "correction", "trace_id": "tut-01-fix"})
```

`sdk.ref(Cls, **identity)` gets a managed reference for the entity. `sdk.set` is the low-level write — it writes one assertion and returns its id (we don't need the id here). The metadata distinguishes this assertion from the original; it'll show up that way in any audit package.

Re-read:

```python
print(sdk.get(Person, person_id="p-001", locale="en").name)
# Alicia
```

The snapshot shows the new value. The original assertion is still in the ledger — we didn't overwrite anything, we appended a new entry.

## Looking at the history

The snapshot exposes a per-field assertions namespace that lets us inspect what's been asserted, not just what reduced to the snapshot:

```python
snap = sdk.get(Person, person_id="p-001", locale="en")

print("active assertions for name:", len(snap.assertions.name.active))
# 1 — only the latest reduces into the snapshot

print("history of name:", len(snap.assertions.name.history))
# 2 — both "Alice" and "Alicia" are in the ledger

for asrt in snap.assertions.name.history:
    print("  ", asrt.value, asrt.meta.get("source"))
# Alice    tutorial
# Alicia   correction
```

This is the audit story in miniature: both assertions are visible, with their sources, in order. If you needed to know *who said what when*, the answer is here without anything special being set up.

## Retracting an assertion

What if instead of *correcting* a name, we wanted to express that an assertion *should never have been made*? That's what retraction is for. It doesn't remove the assertion from the ledger; it appends a new entry that causes future projections to skip the original.

Suppose Alice's "early-adopter" tag was added by mistake:

```python
# Find the assertion id for the bad tag
snap = sdk.get(Person, person_id="p-001", locale="en")
bad_assertion = next(
    a for a in snap.assertions.tag.active
    if a.value == "early-adopter"
)

# Retract it
sdk.retract(bad_assertion.asrt_id,
            meta={"source": "correction", "trace_id": "tut-01-fix-2"})
```

Re-read:

```python
print(sdk.get(Person, person_id="p-001", locale="en").tag)
# {"vip"}  — "early-adopter" no longer reduces in
```

But the history still has it:

```python
snap = sdk.get(Person, person_id="p-001", locale="en")
for asrt in snap.assertions.tag.history:
    print("  ", asrt.value, asrt.retracted)
# vip              False
# early-adopter    True
```

The audit reader sees both: the assertion happened, and the retraction happened. *"This was claimed and then withdrawn"* is a different story from *"this was never claimed"*, and the ledger preserves the difference.

(Use the `correction` pattern for *"the value changed"*, and `retract` for *"the original was a mistake"*. Both are append-only, both honest about what happened — they just tell different stories to whoever reads the audit.)

## Persisting to disk

So far everything has been in memory. To survive a restart, open the store with a `ledger_path`:

```python
sdk_persistent = SDKStore.from_schema_classes(
    classes,
    ledger_path="./tutorial.factpy.db",
)
```

The first time you run this, the file is created and the schema's digest is written into it. Re-run the same code with the same schema and the file opens cleanly. Change the schema in a way that makes old facts unreadable (rename a field, change a cardinality), and the open will fail with a clear schema-mismatch error.

We won't write to `sdk_persistent` in this tutorial — we leave that for [tutorial 02](02-rules-and-derivations.md), where the persistence will matter. For now, the in-memory store is fine. The [persistence guide](../guides/persistence.md) covers the topic in detail when you need it.

## Recap

In ~50 lines you've:

- declared a typed schema with three entities, including a composite identity and a relationship,
- validated the schema before opening,
- opened an in-memory store and wrote a batch of facts atomically,
- read entities by identity and filtered them by field,
- corrected a wrong value with a new assertion,
- retracted a mistaken assertion,
- inspected the per-field history to see both the snapshot and the audit story,
- learned how to point the same code at a persistent file when needed.

Everything from here builds on this surface. The next tutorial introduces the rule DSL and the derivation lifecycle — the part of factpy that *infers* facts rather than just storing the ones you write.

## Where to next

- [Tutorial 02 — Rules and derivations](02-rules-and-derivations.md) — querying the ledger, named rules, candidates and accepts.
- [Reading and writing guide](../guides/reading-and-writing.md) — for the ergonomic reference of every move covered above.
- [The ledger concept page](../concepts/the-ledger.md) — for the conceptual model under all of this.