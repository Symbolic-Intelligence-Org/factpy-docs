---
title: "Entities and Fields"
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

# Entities and fields

If facts are the atomic claims factpy stores, the schema is the *vocabulary* in which those claims are made. It says: *these are the kinds of things we make claims about, these are the kinds of claims we make about them, and this is how a claim is addressed.*

A schema in factpy is not a table definition. There is no migration, no shape that "rows" must conform to, no field that "must" have a value. The schema constrains the *grammar* of writes and reads — what predicates exist, how they are typed, how they reduce — without constraining what the world has to look like.

This page covers the pieces of that vocabulary: entities and the identity coordinates that locate them, fields and the cardinality rules that govern them, type domains, and how schemas are composed and evolved.

## Entities

An **entity** declares a kind of thing the system can hold facts about. You declare it as a class:

```python
from kernel.sdk import Entity, Identity, Field

class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")
```

The class is not instantiated to *create* a person; nothing in factpy creates anything. The class describes what facts about a `Person` can look like and how to address them. Writing a fact about a particular person is a separate operation that takes the entity class and identity coordinates as a lookup key.

Two members of the class do real work: `Identity` declarations tell factpy how to *locate* an entity, and `Field` declarations tell factpy what *predicates* can carry facts about it.

## Identity

An identity is a coordinate. It is how the ledger and the SDK know *which* entity a particular fact is about.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
```

`primary_key=True` marks the field as a coordinate in the entity's address. An entity can have more than one identity field, and the combination must be unique:

```python
class User(Entity):
    user_id: str = Identity(primary_key=True)
    locale: str = Identity()
```

A `User` is now addressed by the *pair* `(user_id, locale)`. Two users with the same `user_id` but different `locale` values are different entities, with their own histories of facts. This composite-identity pattern is common when an entity is naturally scoped — by tenant, by region, by language — and you want that scoping to be part of how the entity is named, not a field on it.

Identities can declare a default or a factory:

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
```

`default_factory="uuid4"` lets the SDK assign a fresh identity when one is not supplied. `default=...` provides a fixed default. Most domain entities have a meaningful natural key (an order id, a person id, an iso code) and do not need either; surrogate-keyed entities like edges between other entities tend to use `default_factory`.

Identity values are *not* facts. They are not stored as ledger entries, they are not reducible, and they cannot be retracted. They live on the entity's address. Everything else — including which country a person lives in — is a fact, and lives in the ledger.

## Fields

A **field** is a predicate: a kind of claim the schema permits. Each field declaration sets a cardinality and an underlying type.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")
    age: int = Field(cardinality="single")
```

`cardinality="single"` says: the snapshot of this entity has at most one current value for this field. Multiple assertions over time are allowed and the ledger keeps them all, but only the latest non-retracted assertion shows up in the snapshot.

`cardinality="multi"` says: the snapshot accumulates every asserted-and-not-retracted value into a set. Multiple assertions of the same value collapse to one entry in the snapshot, even though every assertion is preserved as its own ledger entry.

The cardinality is a property of the *snapshot reduction*, not a property of the ledger. The ledger does not care; it stores every assertion. The reduction rule is what determines whether you see one value or many when you ask *what is the current state*.

Fields can carry an optional `description=` for inline documentation that flows into authoring artifacts and audit packages.

## Type domains

The python annotation on a field is mapped to one of factpy's type domains:

`string`, `int`, `bool`, `bytes`, `float64`, `uuid`, `time` (a `datetime`), and `entity_ref` (a reference to another entity). The annotation `str` becomes `string`, `int` stays `int`, `UUID` becomes `uuid`, `datetime` becomes `time`, and any annotation that is itself an `Entity` subclass becomes `entity_ref`:

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
    home_country: "Country" = Field(cardinality="single")
```

A field of type `entity_ref` is what lets rules join across entities. *"Find every person whose home country is Germany"* is expressible because `home_country` is itself a fact whose value is the address of another entity.

The type domain is part of the schema. It travels with the field into the ledger and into rule compilation, and it is part of the schema digest used for persistence checks.

## Relationships as entities

In practice, factpy does not require a separate relationship construct for most modelling. A relationship between two entities is itself an entity, with each end as an `entity_ref` field:

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
```

`LivesIn` is an entity whose facts happen to refer to two other entities. It can carry its own attributes (`since`), accumulate its own history, and be the subject of rules in exactly the same way as any other entity. Querying *"every person who lives in Germany since 2020"* is a join on `LivesIn` instances under the usual rule language.

For systems that need typed edges as a first-class kernel construct — a discrete `Relationship` declaration with explicit `from_entity` and `to_entity` types — the kernel provides a `Relationship` base class. Most application code does not need it; the entity-as-relationship pattern above is sufficient.

## Entity metadata

Each entity can declare a small amount of metadata via an inner `Meta` class:

```python
class Person(Entity):
    """A natural person known to the system."""

    class Meta:
        version = "1.0.0"
        description = "Natural person"
        tags = ["domain:identity"]

    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
```

`version` is part of the schema digest and travels with rule and derivation runs. `description` falls back to the class docstring if not given. `tags` are free-form labels useful for organizing schemas in larger systems.

`Meta` accepts only `version`, `description`, and `tags`. Anything else raises at class-definition time, on purpose: schema declarations are part of the audit story, and the kernel keeps that surface narrow.

## Schemas, ledgers, and the digest

When you open a persistent ledger, factpy compares the schema you opened it with against a digest stored in the file. If a field's cardinality, type domain, or identity composition has changed in a way that would make existing facts unreadable, the open fails loudly.

This is what makes the schema *part of* the ledger's contract rather than separate from it. The facts in the ledger are only meaningful through the lens of the schema that wrote them; changing that lens silently would break replay, make audit packages misleading, and let derived state quietly diverge from its source.

Adding a new field, adding a new entity, or relaxing optional metadata is fine. Renaming a field, changing a cardinality, or changing a type domain is a *new schema version*, and existing ledgers written under the old schema must be migrated explicitly rather than reinterpreted in place.

See [the ledger](the-ledger.md) for how the digest is computed and where it lives.

## Composing schemas

Entities are normal python classes, declared in normal modules, imported normally. A schema is just *the set of entity classes you hand to the SDK at open time*:

```python
from kernel.sdk import SDKStore
from myapp.schema.identity import Person, Country
from myapp.schema.commerce import Order, LineItem

sdk = SDKStore.from_schema_classes([Person, Country, Order, LineItem])
```

There is no central registry, no schema file, no decorator that has to run for an entity to "exist." Splitting a schema across modules — by domain, by ownership, by team — is the same act as splitting any other python codebase. Combining two schemas is the same act as importing from two packages.

This matters more than it sounds. It means a schema is something you can refactor, compose, vendor, or extract without operational ceremony. The kernel only ever sees the list of classes it was given.

## Where to next

- [The ledger](the-ledger.md) — how facts asserted under this schema are stored and reduced.
- [Rules and derivations](rules-and-derivations.md) — how the schema's predicates show up in queries and inference.
- [Audit and provenance](audit-and-provenance.md) — how the schema, including its version, is captured in audit packages.
- The [Defining a schema guide](../guides/defining-a-schema.md) (when written) walks through real-world schema patterns: composite identities, surrogate keys, evolving a schema across versions.