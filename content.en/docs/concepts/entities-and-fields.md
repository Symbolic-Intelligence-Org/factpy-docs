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

If facts are the atomic claims factpy stores, the schema is the vocabulary in which those claims are made. It declares what kinds of thing the system can hold facts about, what kinds of claim are admissible against them, and how a particular claim is addressed to its subject. A schema is not a table definition: it imposes no layout on storage, requires no field to carry a value at any given moment, and says nothing about what the world has to look like. What it constrains is the grammar of writing and reading — the set of predicates that exist, the type domains they range over, and the rules under which multiple assertions to the same predicate reduce. The remainder of this page describes the components of that grammar in turn, together with how schemas are composed across modules and how they evolve from one version to the next.

## Entities

An entity declares a kind of thing about which the system can hold facts. The declaration takes the form of a python class:

```python
from kernel.sdk import Entity, Identity, Field

class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")
```

The class is never instantiated to bring a person into being; nothing in factpy is brought into being by class instantiation, since the kernel records facts in a ledger and the class is only the schema's description of which facts about a `Person` are admissible and how a particular person is to be addressed when a fact is to be written about her. Two kinds of declaration carry the substance of an entity: identity declarations, by which the kernel locates a particular instance, and field declarations, which fix the predicates available for assertion against it.

## Identity

An identity is a coordinate by which the ledger and the SDK locate the entity that a fact is about.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
```

The flag `primary_key=True` marks the field as part of the entity's address. An entity may have more than one such field, in which case the combination of values must be unique:

```python
class User(Entity):
    user_id: str = Identity(primary_key=True)
    locale: str = Identity()
```

A `User` is then addressed by the pair `(user_id, locale)`, and two users sharing a `user_id` but differing in `locale` are distinct entities with separate histories of facts. The composite-identity pattern is appropriate wherever an entity is naturally scoped — by tenant, by region, by language — and the scoping properly belongs to the entity's name rather than to a field on it. Where two such instances should be able to coexist with independent histories, the distinguishing value belongs in the identity rather than in a field.

An identity may declare a default value or a factory for cases in which a value is not supplied at write time:

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
```

`default_factory="uuid4"` instructs the SDK to assign a fresh identifier when the caller supplies none, while `default=...` provides a fixed value used in its place. Most domain entities have a meaningful natural key — an order id, an iso code, a person id — and require neither; surrogate-keyed entities, typically those that record edges between other entities, are the usual occasion for `default_factory`.

Identity values are not themselves facts. They are not stored as ledger entries, they are not subject to projection or reduction, and they cannot be retracted, because they live on the entity's address rather than within its history. Everything else, including the country a person lives in or the address of a corporate residence, is held as a fact in the ledger.

## Fields

A field is a predicate, that is, a kind of claim that the schema admits. Each declaration fixes the field's cardinality and its underlying type domain.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")
    age: int = Field(cardinality="single")
```

A cardinality of `"single"` specifies that the snapshot of an entity carries at most one current value for the field at any moment. The ledger continues to accept and to retain any number of assertions to that field over time, but the projection that produces the snapshot includes only the latest non-retracted entry. A cardinality of `"multi"` specifies that the snapshot accumulates every active assertion into a set, with multiple assertions of the same value collapsing to a single entry in the projection while remaining distinct in the ledger.

Cardinality is a property of the projection, not of the ledger itself. The ledger stores every assertion regardless of cardinality, and the reduction rule is what determines, at the moment a snapshot is computed, whether one value or many is exposed. A field may also carry an optional `description` string, which propagates into authoring artifacts and audit packages as inline documentation of the predicate.

## Type domains

The python annotation on a field is mapped onto one of the kernel's type domains. The available domains are `string`, `int`, `bool`, `bytes`, `float64`, `uuid`, `time` (corresponding to a `datetime`), and `entity_ref` (a reference to another entity). The annotation `str` becomes `string`, `int` remains `int`, `UUID` becomes `uuid`, `datetime` becomes `time`, and any annotation that is itself a subclass of `Entity` becomes `entity_ref`.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
    home_country: "Country" = Field(cardinality="single")
```

A field declared as `entity_ref` does not carry a value copied from another entity; it carries the address at which that other entity sits in the ledger. This indirection is what enables a rule to walk from one entity to another in the course of evaluation. A query of the form find every person whose home country is Germany works because home_country on a Person holds the reference to a Country, and the rule can follow that reference, read the country's iso_code, and retain only those persons for which it is "DE". The type domain attached to a field is recorded alongside the field's assertions in the ledger, consulted during the compilation of rules that mention the field, and folded into the schema digest that the kernel verifies at open time.

## Relationships as entities

In most modelling situations factpy does not require a dedicated relationship construct: a relationship between two entities is itself an entity, with each end held as an `entity_ref` field.

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
```

`LivesIn` is an ordinary entity whose facts refer to two others. It carries its own attributes — here, the year `since` — accumulates its own history of assertions, and is the subject of rules under the same DSL as any other entity. The query *which persons have lived in Germany since 2020* becomes a join over `LivesIn` instances under the same rule language used for any other join.

For systems that require typed edges as a first-class kernel construct — that is, an explicit `Relationship` declaration carrying `from_entity` and `to_entity` types — the kernel provides a `Relationship` base class. Most application code has no occasion to use it, since the entity-as-relationship pattern above suffices for ordinary modelling; the place where the typed-edge form earns its weight is in adapters such as PyReason, where the engine itself works on a typed graph rather than on entities and predicates.

## Entity metadata

An entity may carry a small amount of metadata through an inner `Meta` class:

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

The `version` field is part of the schema digest and travels into rule and derivation runs, where it lets an audit reader distinguish results obtained under one version of an entity's schema from those obtained under another. The `description` field, when omitted, defaults to the class docstring; the `tags` are free-form labels useful for organising large schemas across teams or domains. `Meta` accepts only these three keys, and any further key raises at class-definition time. The narrowness is deliberate: schema declarations are themselves part of the audit story, and the kernel keeps the surface that participates in that story small enough to remain predictable.

## Schemas, ledgers, and the digest

When a persistent ledger is opened, the kernel compares the schema supplied at open time against a digest stored in the ledger file. A change in any field's cardinality, type domain, or identity composition that would render existing assertions unreadable causes the open to fail with an explicit error rather than to proceed with a silent reinterpretation. This is what makes the schema part of the ledger's contract rather than an external commentary on it: the facts in the ledger are meaningful only under the schema that wrote them, and a silent change to that schema would invalidate replay, would render audit packages misleading, and would allow derived state to diverge from the assertions on which it was nominally based.

Additions are unproblematic. A new field, a new entity, or the addition of optional metadata where none existed previously preserves legibility and does not affect the digest. Renaming a field, changing a cardinality, or changing a type domain does not preserve legibility, constitutes a new schema version, and requires that any ledger written under the previous schema be migrated explicitly rather than reinterpreted in place. The mechanics of the digest, the distinction between legible and illegible changes, and migration as a replay between two ledgers are taken up in [the ledger](the-ledger.md) and in the [persistence guide](../guides/persistence.md).

## Composing schemas

Entity classes are ordinary python classes declared in ordinary modules and imported as any other class would be. A schema, from the kernel's perspective, is simply the list of entity classes supplied to the SDK at the moment a store is opened.

```python
from kernel.sdk import SDKStore
from myapp.schema.identity import Person, Country
from myapp.schema.commerce import Order, LineItem

sdk = SDKStore.from_schema_classes([Person, Country, Order, LineItem])
```

There is no central schema registry, no separate schema file, and no decorator whose execution is required for an entity to be recognised. Splitting a schema across modules — by domain, by ownership, by team — is the same act as splitting any other python codebase, and combining two schemas is the same act as importing from two packages. The architectural consequence is that a schema can be refactored, composed, vendored, or extracted without further ceremony, because the kernel sees only the list of classes it has been given and treats them as the schema for the duration of the session.

## Where to next

[The ledger](../the-ledger) develops the storage and projection layer that the schema's predicates write into. [Rules and derivations](../rules-and-derivations) develops the reasoning layer in which those predicates appear. [Audit and provenance](../audit-and-provenance) describes how the schema, including its version, is captured in audit packages. The [defining a schema guide](../../guides/defining-a-schema) walks through real-world schema patterns — composite identities, surrogate keys, and evolving a schema across versions — for readers who want the same material in task-oriented form.