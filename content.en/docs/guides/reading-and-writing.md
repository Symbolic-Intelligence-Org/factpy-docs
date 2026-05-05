---
title: "Reading and Writing"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Reading and writing

This guide covers the day-to-day mechanics of putting facts into the ledger and getting them back out. It assumes you have a schema and an open store. For the schema side, see [defining a schema](defining-a-schema.md). For the conceptual model — assertions, snapshots, projections — see [the ledger](../concepts/the-ledger.md).

factpy gives you two paths for writes: a **batch API** that groups multiple writes into one transactional unit, and a **direct API** that writes one assertion at a time and returns its id. They produce identical entries in the ledger; they differ only in ergonomics.

## The batch path

The batch API is what most application code uses. You open a transaction, build up entities and writes against it, optionally preview the plan, and commit:

```python
tx = sdk.batch(meta={"trace_id": "seed-001", "source": "import_job"})

de = tx.entity(Country, iso_code="DE")
de.name.set("Germany")

alice = tx.entity(Person, person_id="p-001", locale="en")
alice.name.set("Alice")
alice.tag.add("vip")
alice.home_country.set(de)

result = tx.commit()
print("written assertions:", len(result.apply_result.assertion_ids))
```

A few things are happening here. `sdk.batch(meta=...)` opens the transaction; the `meta` flows onto every assertion the transaction writes, unless you override it per-call. `tx.entity(Cls, **identity)` returns a handle on which you can call `.field.set(...)` and `.field.add(...)`. Handles know the entity's address and the schema's predicates, so the writes are typed and validated locally before they're queued.

Cross-entity references work by passing one handle as another's value:

```python
alice.home_country.set(de)
```

The handle stands in for the entity's address; the SDK resolves it to a managed reference at commit time. You don't construct reference strings by hand and the API rejects ones that didn't come from `tx.entity` or `sdk.ref`.

Before committing, you can preview the plan to see what will happen without writing:

```python
plan = tx.preview()
print("planned ops:", len(plan.ops))
```

`tx.commit()` applies everything atomically: either every assertion in the batch makes it into the ledger or none do. Errors raise before any writes are visible.

### Partial identity

When an entity has a composite identity and only part of it is known up front, declare what you have and bind the rest later:

```python
bob = tx.entity(Person, person_id="p-002")
bob = bob.bind(locale="en")
bob.name.set("Bob")
```

`bind` returns a fully-identified handle. Calling field methods on a partially-identified handle raises rather than silently writing to the wrong address.

## The direct path

When you want fine-grained control — explicit assertion ids, per-assertion metadata, retractions of specific historical writes — drop down to the direct API:

```python
ref = sdk.ref(Person, person_id="p-001", locale="en")

asrt_alias = sdk.add(Person.alias, ref, "Alice Cooper",
                     meta={"source": "review", "trace_id": "rev-44"})
asrt_age   = sdk.set(Person.age, ref, 31,
                     meta={"source": "review", "trace_id": "rev-44",
                           "version": "v2", "valid_from": "2024-01-01T00:00:00+00:00"})
revoker    = sdk.retract(asrt_alias,
                         meta={"source": "review", "trace_id": "rev-45"})
```

`sdk.ref(Cls, **identity)` registers the identity and returns a managed reference string you can pass to subsequent calls. `sdk.set` and `sdk.add` return the assertion id of the write — useful when you want to record exactly which assertion was made (for instance, in a downstream system that will later ask to retract it). `sdk.retract` takes an assertion id, not a value, and returns the retraction's own id (or `None` if the target was already retracted).

Use the direct API when:

- you need the assertion id of a specific write to retract or reference later,
- you want different metadata on each call,
- you're translating from another system's stream of events one at a time.

For everything else, the batch API is shorter and atomic.

## Reading single entities

The simplest read is a snapshot of one entity:

```python
alice = sdk.get(Person, person_id="p-001", locale="en")
print(alice.name)        # "Alice"
print(alice.tag)         # {"vip"}
print(alice.home_country) # the Country snapshot
```

`sdk.get(Cls, **identity)` returns a read-only snapshot — the projection of the ledger at the chosen view. Single-cardinality fields read as values; multi-cardinality fields read as sets; `entity_ref` fields read as snapshots of the referenced entity.

When you need not just the current value but its history, the snapshot exposes a per-field assertions namespace:

```python
print(alice.assertions.age.active)      # current assertions
print(alice.assertions.age.history)     # all assertions ever
print(alice.assertions.age.version("v2"))  # filtered by meta version
```

`active` is what reduced into the snapshot. `history` is everything in the ledger for that field on that entity. `version(...)` filters by a `version` meta key — a useful pattern when you record multiple parallel versions of a value (an old and a new tax rate, a draft and a published price).

If the entity does not exist in the active view — no `<T>:exists` assertion in scope — `sdk.get` returns `None` rather than a snapshot of nothing.

## Reading by filter

To find entities that match an identity-and-field filter, use `sdk.find`:

```python
vips = sdk.find(Person, tag="vip", limit=50)
for row in vips:
    print(row.ref, row.name)
```

`find` returns a list of rows shaped to the entity. Each row has a `ref` attribute (the managed reference string) and the field values you can read like attributes. This is the ergonomic way to enumerate entities matching a coarse filter; for richer queries — joins, negation, sub-rules — use a `Rule` and `sdk.run` (see [writing rules](writing-rules.md), when written).

## Metadata, by convention

`meta` accepts arbitrary string-keyed values. Some keys are conventional and useful to standardize across your codebase:

`source` identifies the system or process that produced the assertion (`"import_job"`, `"reviewer"`, `"derivation:drv.vip_inference"`). `trace_id` ties multiple assertions to one logical operation — invaluable when reading audit packages later. `version` lets you carry parallel versions of a value (`"v1"`, `"v2"`); the snapshot's `assertions.<field>.version("v2")` reads them back. `valid_from` and `valid_to` carry semantic time when distinct from physical assertion time. `confidence` is conventional for derived or model-produced facts.

None of these are *required*. The kernel only requires that `meta` be a flat dict of JSON-friendly values. The conventions above are what most factpy codebases settle on; following them makes audit packages easier to navigate later.

## Retraction is a new entry

A retraction does not modify or remove the original assertion. It is a new ledger entry that, when the snapshot is computed, causes the original assertion to be skipped during reduction. The original assertion is still in the ledger, still queryable through `assertions.<field>.history`, still part of any audit package.

That means there is no such thing as "fixing" a fact in place. To correct a value, write a new assertion (the snapshot will use the latest); to express that a previously-asserted fact was wrong, retract it (the snapshot will skip it but the audit trail will show that someone said so, when, and why).

When in doubt, prefer a new assertion to a retraction. Retractions are for *"this should not have been asserted"*; new assertions are for *"the truth has changed"*. Both are append-only, both are honest about what happened; the difference is a story you tell the audit reader.

## Idempotence and re-runs

Re-running a write that has already happened is *not* a no-op. It produces a new ledger entry — same predicate, same subject, same value, but a new assertion id, a new timestamp, and (potentially) different metadata. Snapshots will look the same; the ledger will have grown by one entry.

For a `multi`-cardinality field, this is harmless: the snapshot collapses duplicate values to one. For a `single`-cardinality field, it is also harmless in terms of the snapshot, but the ledger now records that the same value was re-asserted at a later time — which can itself be useful information ("this value was confirmed again on 2024-04-01 by the import job").

If you need true idempotence — *"only write if not already present"* — wrap the write in a read-then-write check, or accept that re-runs will accumulate equivalent entries and let the audit story carry that.

## Errors worth knowing

`SDKSchemaError` is raised when you read or write something the schema doesn't permit — an unknown field, missing identity values, a type-domain mismatch on a value.

`CardinalityError` is raised when you call `set` on a `multi` field or `add` on a `single` field. The cardinality is part of the field declaration; the error tells you which of the two you should have used.

`EntityNotFoundError` is raised by `sdk.edit(...)` when no such entity exists. (`sdk.get(...)` does not raise; it returns `None`.)

`SDKStoreError` is the catch-all for store-level problems — bad references, schema-digest mismatches at open time, and structural issues with the call.

Each error carries a `code` attribute (e.g. `"FIELD_CARDINALITY_MISMATCH"`, `"UNRESOLVABLE_E_REF"`) that is stable across versions. The full list lives in the [error codes reference](../reference/error-codes.md) (when written).

## Where to next

- The [Writing rules guide](writing-rules.md) (when written) covers querying the ledger with the rule DSL — joins across entities, sub-rules, negation patterns.
- The [Persistence guide](persistence.md) (when written) covers ledger files, the schema digest, and crossing process boundaries.
- For the conceptual underpinning of every read and write covered here, see [the ledger](../concepts/the-ledger.md).