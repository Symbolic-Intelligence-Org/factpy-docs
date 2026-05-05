---
title: "Defining a Schema"
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

# Defining a schema

This guide develops the practical decisions that arise when writing a schema for a real domain: where the schema lives in a project, how identities are chosen, how cardinalities are picked, how relationships are modelled, how schemas are split across modules, how they are validated before a store is opened, and how they evolve from one version to the next. It assumes familiarity with the conceptual material developed in [entities and fields](../../concepts/entities-and-fields), and concentrates here on the choices that have to be made when one sits down to declare a schema for the first time.

## Where the schema lives

A schema is a collection of `Entity` subclasses, declared in ordinary python modules and imported as any other class would be. The kernel maintains no central registry, requires no manifest file, and depends on no decorator that has to run for an entity to be recognised. A typical project organises its schema under a `schema/` package alongside the application code, with related entities grouped into modules.

```
myapp/
├── schema/
│   ├── __init__.py
│   ├── identity.py      # Person, Country
│   └── commerce.py      # Order, LineItem
└── app.py
```

Whatever set of classes is supplied to `SDKStore.from_schema_classes(...)` at the moment the store is opened constitutes the schema for the duration of the session, and the kernel sees nothing beyond that list.

## Choosing an identity strategy

The first decision for any entity is how it will be addressed, and three patterns cover almost every case that arises in practice.

The first is the *natural single key*, used wherever the domain itself supplies a meaningful unique identifier. An iso code uniquely identifies a country, an order id uniquely identifies an order, and there is no need to invent further structure.

```python
class Country(Entity):
    iso_code: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
```

`iso_code` is the address; `Country(iso_code="DE")` and `Country(iso_code="FR")` are different entities, with separate histories.

The second is *composite identity*, used wherever an entity is naturally scoped — by tenant, by region, by language — and the scope properly belongs to the entity's name rather than to a field on it. The criterion is whether two such instances should be able to coexist with independent histories: when they should, the distinguishing value belongs in the identity rather than in a field.

```python
class User(Entity):
    user_id: str = Identity(primary_key=True)
    locale: str = Identity()
    name: str = Field(cardinality="single")
```

The user is now addressed by the pair `(user_id, locale)`, and `User(user_id="u-1", locale="en")` and `User(user_id="u-1", locale="de")` are distinct entities with separate histories of facts. The pattern is appropriate when the scope is a real partition of the data — when the two locales of one user really are two things, with their own names, their own tags, and their own audit trails — and inappropriate when the scope is simply an attribute that happens to vary over time.

The third is the *surrogate key with a factory*, used wherever there is no meaningful natural key. The typical case is an entity that exists purely in order to relate other entities to one another: a residence, an authorship, a review assignment.

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
```

The `default_factory="uuid4"` instructs the SDK to assign a fresh identifier whenever none is supplied at write time. The available factories are documented in the reference; `uuid4` is the common one, and is sufficient for nearly all surrogate-key situations. The decision rule between the three patterns reduces to a single question: if removing the field would leave the entity ambiguous in the domain, the field belongs in `Identity`; otherwise it belongs in `Field`.

## Choosing cardinality

Each `Field` is declared as either `cardinality="single"` or `cardinality="multi"`, and the choice affects what the snapshot exposes rather than what the ledger stores. Both cardinalities preserve every assertion in the ledger; they differ only in how the projection reduces multiple assertions to a presented value.

A single-cardinality field is appropriate wherever the entity has at most one current value at any moment. A name, a home country, a date of birth — each is one thing about the entity, and a new assertion supersedes the previous one in the projection.

```python
name: str = Field(cardinality="single")
home_country: Country = Field(cardinality="single")
date_of_birth: datetime = Field(cardinality="single")
```

A multi-cardinality field is appropriate wherever the entity carries a *set* of values that coexist. A tag, an alias, a preferred language — each is something an entity may have any number of, all simultaneously valid.

```python
tag: str = Field(cardinality="multi")
alias: str = Field(cardinality="multi")
preferred_language: Language = Field(cardinality="multi")
```

Multiple `add` calls with the same value collapse to one entry in the snapshot but remain distinct entries in the ledger, which is what allows the question of *how many times this tag was asserted, and by whom, and when* to be answered from the audit trail without affecting what the snapshot shows. The general guidance is that a field holding a small fixed number of distinct values — a status, a primary contact — is single-cardinality, while a field holding an open-ended set is multi-cardinality.

## Modelling relationships

A connection between entities can be modelled in two ways, and the choice depends on whether the relationship has anything to say about itself or merely connects one thing to another.

The simpler form is a *direct entity reference*, declared as an `entity_ref` field on the source entity. This is appropriate whenever the relationship is purely directional and carries no attributes of its own.

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    home_country: Country = Field(cardinality="single")
```

Asserting `home_country` records one fact: a single predicate, with the country's address as the value. The relationship is queryable, the projection straightforward, and no further entity is needed to carry the connection.

The richer form is a *relationship as entity*, used whenever the connection has attributes, a lifecycle, or a history of its own. An entity is declared whose fields are the endpoints of the relationship together with whatever else the relationship needs to carry.

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
    until: int = Field(cardinality="single")
```

The relationship is now a first-class subject in the schema. It accumulates its own provenance, can carry its own facts (`since`, `until`), and can be the subject of rules in exactly the way any other entity can. Multiple `LivesIn` instances can exist for the same person, which is the appropriate model for someone who has lived in more than one country over time. The decision criterion reduces to a single question: does the relationship have anything to say about itself? If it does, it deserves an entity; if it does not, an `entity_ref` field on the source is sufficient.

## Entity metadata

An inner `Meta` class allows a small amount of metadata to travel with the entity into authoring artifacts and audit packages.

```python
class Person(Entity):
    """A natural person known to the system."""

    class Meta:
        version = "1.0.0"
        description = "Natural person"
        tags = ["domain:identity", "owner:platform"]

    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
```

The `version` field participates in the schema digest, so that a deliberate version bump is visible to anyone opening a ledger written under a previous version. The `description`, when omitted, defaults to the class docstring. The `tags` are free-form string labels useful for grouping entities by domain, ownership, or compliance scope in larger schemas. `Meta` accepts only these three keys, and any other key raises at class-definition time; the narrowness is intentional, since schema declarations are themselves part of the audit story and the surface that participates in that story is kept small.

## Splitting across modules

Schemas grow, and they may be split along whatever axis the project finds natural — by domain, by ownership, by architectural layer.

```python
# myapp/schema/identity.py
class Person(Entity):
    ...

class Country(Entity):
    ...

# myapp/schema/commerce.py
from myapp.schema.identity import Person

class Order(Entity):
    order_id: str = Identity(primary_key=True)
    customer: Person = Field(cardinality="single")
    ...
```

The schema is assembled at the moment the store is opened, by importing the relevant classes and supplying them as a single list.

```python
from myapp.schema.identity import Person, Country
from myapp.schema.commerce import Order, LineItem

sdk = SDKStore.from_schema_classes([Person, Country, Order, LineItem])
```

There is no further ceremony involved in splitting and recombining: an entity referenced through `entity_ref` in another module need only be importable from where it is referenced, and the kernel has no further interest in where a class was declared. For a vendored schema or a schema published as a library, the entity classes are typically exposed from a top-level `__init__.py`, allowing consumers to compose them with classes of their own.

## Validating before opening

`schema_preflight_from_classes` runs the same compilation that `from_schema_classes` performs and returns a structured report without touching any ledger.

```python
from kernel.sdk import schema_preflight_from_classes

report = schema_preflight_from_classes([Person, Country, Order])
```

A missing `primary_key`, a circular reference, a type-domain mismatch — each surfaces as a structured entry in the report rather than as a runtime exception during the first write. Running the preflight in a test suite or in continuous integration is the simplest way to catch schema regressions early, before they reach a ledger that depends on the schema being well-formed.

## Opening the store

A store can be opened in either of two modes. The in-memory mode is appropriate for tests and short-lived computation: the ledger exists for the lifetime of the process and is discarded when the process exits.

```python
from kernel.sdk import SDKStore

sdk = SDKStore.from_schema_classes([Person, Country, Order])
```

The persistent mode, opt-in through a `ledger_path`, is appropriate for any long-running service or for state that needs to survive a restart.

```python
sdk = SDKStore.from_schema_classes(
    [Person, Country, Order],
    ledger_path="./data/factpy.db",
)
```

The first time a store is opened with a given path, the schema's content digest is recorded in the file. On every subsequent open, the digest of the schema being supplied is recomputed and compared against the stored one. A change in any field's cardinality, type domain, or identity composition that would render existing assertions illegible causes the open to fail with an explicit schema-mismatch error rather than to proceed with a silent reinterpretation. This check is the principal guard against unintentional schema drift on a persistent ledger, and it should be respected wherever it triggers.

## Evolving a schema

Some changes preserve the schema's ability to read existing assertions and require no version bump and no migration: adding a new entity, adding a new field to an existing entity, relaxing optional metadata, and adding a new `Identity` field that has either a `default` or a `default_factory`. The common property is that no assertion already in the ledger is rendered illegible by the change; the additions extend the vocabulary without altering the meaning of what was previously written.

Other changes do not preserve legibility and constitute a new schema version: renaming a field or an entity, changing a field's cardinality between single and multi, changing a field's type domain, altering the identity composition of an existing entity, and removing a field that has been written to. The kernel refuses to open a ledger written under a different schema digest, and migration is therefore an explicit operation rather than an implicit reinterpretation: the entity's `Meta.version` is bumped, a migration program replays the old ledger into a new one under the new schema, and the new ledger is opened only after the migration has run. For first-week development against a still-molten schema, with no ledger that depends on continuity, deleting the file and reopening is the appropriate course; the digest check exists to protect data that exists, and where there is no such data it imposes no further constraint.

## Where to next

The [reading and writing guide](../reading-and-writing) covers the day-to-day mechanics of `set`, `add`, `retract`, batches, and write-time metadata. The [SDK reference](../reference/sdk) lists every entry point with its full signature and options. For the conceptual background on which these practical decisions rest, see [entities and fields](../../concepts/entities-and-fields) and [the ledger](../../concepts/the-ledger).