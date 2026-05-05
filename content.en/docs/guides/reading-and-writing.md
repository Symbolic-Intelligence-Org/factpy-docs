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

This guide covers the day-to-day mechanics of putting facts into the ledger and reading them back out: the two write paths the SDK provides, the shape of single-entity and filter-based reads, the conventions around write-time metadata, the semantics of retraction and idempotent re-runs, and the typed errors raised when something is wrong with the call. It assumes a schema and an open store; for the schema side, see [defining a schema](/docs/guides/defining-a-schema), and for the conceptual underpinning of writes, projections, and retractions, see [the ledger](/docs/concepts/the-ledger).

The SDK provides two paths for writes. A *batch API* groups multiple writes into a single transactional unit and is the path most application code uses. A *direct API* writes one assertion at a time and returns its id, which is appropriate when a particular write needs to be addressed individually after the fact. Both paths produce identical entries in the ledger; the difference is one of ergonomics and of when the kernel applies the writes.

## The batch path

A batch transaction is opened with `sdk.batch`, populated with entity handles and field operations, and committed atomically.

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

The `meta` argument supplied to `sdk.batch` flows onto every assertion the transaction writes, unless overridden on a per-call basis. `tx.entity(Cls, **identity)` returns a handle on which field operations are queued: `.field.set(...)` for single-cardinality fields, `.field.add(...)` for multi-cardinality fields. Handles know the entity's address and the schema's predicates, so writes are typed and validated locally before they are queued, and a typing or cardinality mistake is reported at the call site rather than at commit time.

Cross-entity references are passed by handle rather than by string. When `alice.home_country.set(de)` is called with `de` being the handle to a `Country`, the SDK records the country's address as the value of `alice.home_country`. The reference is resolved into a managed string at commit time, and the API rejects values that did not originate as a handle from `tx.entity` or as a managed reference from `sdk.ref`. Reference strings are never to be constructed by hand.

A transaction can be inspected before it is committed. `tx.preview()` returns a plan listing every operation the commit would perform, without writing anything; it is useful when the transaction is constructed dynamically and some validation is needed before the writes become visible.

```python
plan = tx.preview()
print("planned ops:", len(plan.ops))
```

`tx.commit()` applies the queued operations atomically: either every assertion in the batch reaches the ledger or none of them do. Errors raise before any writes become visible, and a partially-applied batch is not a state the kernel allows.

### Partial identity

When an entity has a composite identity and only part of it is known at the moment a handle is created, the handle is declared with what is available and the remaining identity values are bound later through `bind`.

```python
bob = tx.entity(Person, person_id="p-002")
bob = bob.bind(locale="en")
bob.name.set("Bob")
```

`bind` returns a fully-identified handle. Calling field methods on a partially-identified handle raises rather than silently writing to a different address.

## The direct path

The direct API is appropriate wherever a particular write needs to be addressable individually — when the assertion id of a specific write must be retained for a later retraction, when each call carries its own metadata, or when an external stream of events is being translated one assertion at a time.

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

`sdk.ref(Cls, **identity)` registers an identity and returns the managed reference string used by the subsequent calls. `sdk.set` and `sdk.add` return the assertion id of the write, which is the value to retain when the assertion may later need to be referred to or retracted. `sdk.retract` accepts an assertion id rather than a value and returns the retraction's own assertion id, or `None` when the target assertion has already been retracted.

The direct API is the appropriate choice when an individual write needs an individual handle — for retraction by id, for per-call metadata that differs across writes, or for translating an external event stream where each event becomes one assertion. For everything else, the batch API is shorter, atomic by default, and the recommended path.

## Reading single entities

The simplest read is a snapshot of one entity, addressed by its identity coordinates.

```python
alice = sdk.get(Person, person_id="p-001", locale="en")
print(alice.name)         # "Alice"
print(alice.tag)          # {"vip"}
print(alice.home_country) # the Country snapshot
```

`sdk.get(Cls, **identity)` returns a read-only snapshot — the projection of the ledger at the active view, computed on demand from the relevant entries. Single-cardinality fields read as values, multi-cardinality fields as sets, and `entity_ref` fields as snapshots of the referenced entity, which makes navigation across reference fields a matter of attribute access (`alice.home_country.iso_code`).

When the assertion history is needed in addition to the reduced field value, the snapshot exposes a per-field assertions namespace.

```python
print(alice.assertions.age.active)         # current assertions
print(alice.assertions.age.history)        # all assertions ever
print(alice.assertions.age.version("v2"))  # filtered by meta version
```

`active` returns the assertions that reduced into the snapshot. `history` returns every assertion the ledger holds for that field on that entity, retracted entries included, with a flag indicating retraction status. `version(...)` filters by the `version` meta key, which is the conventional shape for carrying parallel versions of a value — an old and a new tax rate, a draft and a published price — alongside one another in the ledger.

A snapshot returned by `sdk.get` is `None` when no entity exists at the supplied identity in the active view (that is, when no `<T>:exists` assertion is in scope). The function does not raise on a missing entity; it returns `None`, and the caller is responsible for handling the absence.

## Reading by filter

For entities matching a coarse identity-and-field filter, `sdk.find` returns a list of entity-shaped rows.

```python
vips = sdk.find(Person, tag="vip", limit=50)
for row in vips:
    print(row.ref, row.name)
```

Each row carries a `ref` attribute, holding the managed reference string, and exposes the entity's fields as attributes. The function is the ergonomic shape for enumerating entities matching simple criteria and is appropriate where the filter can be expressed as field equalities. For richer queries — joins across entities, negation, composition through sub-rules — a `Rule` evaluated by `sdk.run` is the right path; the rule DSL and its mechanics are developed in [writing rules](/docs/guides/writing-rules).

## Metadata conventions

The `meta` argument accepts arbitrary string-keyed values, and the kernel imposes only that the values be JSON-friendly. A small set of keys recurs across factpy codebases by convention rather than by requirement, and adopting them consistently across a codebase makes audit packages easier to navigate later.

The `source` key identifies the system or process responsible for the assertion — `"import_job"`, `"reviewer"`, `"derivation:drv.vip_inference"`. The `trace_id` key ties multiple assertions made under the same logical operation together, which becomes essential when reading audit packages and reconstructing what was done in a single run. The `version` key carries parallel versions of a value, where multiple variants of the same logical fact need to coexist in the ledger; the snapshot's `assertions.<field>.version(...)` reads them back. The `valid_from` and `valid_to` keys carry semantic time when distinct from the physical timestamp of the assertion. The `confidence` key is the conventional shape for facts produced from probabilistic sources or model outputs.

None of these keys is required by the kernel. The conventions above are what most factpy codebases settle on, and following them is the simplest way to ensure that an audit reader, opening a package produced by your system, finds the metadata structured in a familiar way.

## Retraction is a new entry

A retraction does not modify or remove the assertion it targets. It is a new ledger entry that refers to the targeted assertion by its id and that, when the snapshot is computed, causes the targeted assertion to be skipped during reduction. The targeted assertion remains in the ledger, queryable through `assertions.<field>.history`, and forms part of any audit package the run produces. There is no operation in factpy that modifies a fact in place; what looks like correction is always either a new assertion superseding an earlier one in the projection or a retraction recording the explicit withdrawal of the earlier assertion.

The two patterns express different things, and the choice between them matters for the audit story. A new assertion expresses *the truth has changed*: a person's name was Alice, and is now Alicia; both assertions are part of the historical record, and the projection reflects the latest non-retracted value. A retraction expresses *the assertion should not have been made*: someone wrote that this person had the tag `blocked`, and now records that this writing was a mistake; the original assertion is preserved alongside the retraction, and the audit trail makes the act of withdrawal as visible as the act of writing. When in doubt, a new assertion is the appropriate choice; retractions are reserved for the case where the *prior writing* is what is being repudiated.

## Idempotence and re-runs

Re-running a write that has already been performed is not a no-op. It produces a new ledger entry with the same predicate, subject, and value as the previous one, but with its own assertion id, its own timestamp, and potentially different metadata. The snapshot looks the same after the re-run as before; the ledger has grown by one entry.

For a multi-cardinality field, the re-run is harmless to the projection, since the snapshot collapses duplicate values to one. For a single-cardinality field, the snapshot is likewise unchanged, since the new assertion takes the place of the previous one as the latest value while retaining the same value. In both cases, the ledger now records that the value was re-asserted at a later time, which can itself be useful information — *"this value was confirmed again on 2024-04-01 by the import job"* is a meaningful audit-side observation that an overwriting store would not preserve. Where genuine idempotence is required — *"only write if not already present"* — the write is wrapped in a read-then-write check by the application; the alternative is to accept that re-runs accumulate equivalent entries and let the audit story carry the additional records.

## Errors worth knowing

The SDK raises a typed exception hierarchy with stable `code` attributes that downstream code can pattern-match on. `SDKSchemaError` is raised when a read or write references something the schema does not permit — an unknown field, missing identity values, or a type-domain mismatch on a value. `CardinalityError` is raised when `sdk.set` is called on a multi-cardinality field or `sdk.add` is called on a single-cardinality field, since the cardinality is part of the field declaration and the error indicates which of the two should have been called. `EntityNotFoundError` is raised by `sdk.edit(...)` when no entity exists at the supplied identity; `sdk.get(...)`, by contrast, does not raise on a missing entity but returns `None`. `SDKStoreError` is the catch-all for store-level problems — invalid references, schema-digest mismatches at open time, structural issues with the call. Each exception carries a `code` attribute, such as `"FIELD_CARDINALITY_MISMATCH"` or `"UNRESOLVABLE_E_REF"`, that is stable across patch versions; the full set is enumerated in the [error codes reference](/docs/reference/error-codes).

## Where to next

The [writing rules guide](/docs/guides/writing-rules) covers querying the ledger with the rule DSL, including joins across entities, sub-rules, and negation patterns. The [persistence guide](/docs/guides/persistence) covers ledger files, the schema digest, and crossing process boundaries. For the conceptual underpinning of every read and write covered above, see [the ledger](/docs/concepts/the-ledger).