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

This guide walks through writing a schema for a real domain — picking identities, choosing cardinalities, modelling relationships, splitting across modules, and opening it as a store. It assumes you've read [entities and fields](../concepts/entities-and-fields.md) for the concepts, and focuses on the decisions you actually make when sitting down to declare your own.

## Where the schema lives

A schema is a set of `Entity` subclasses. There is no central registry, no manifest file, no decorator that has to run. Put the classes wherever python imports go in your project — typically a `schema/` package next to your application code:

```
myapp/
├── schema/
│   ├── __init__.py
│   ├── identity.py      # Person, Country
│   └── commerce.py      # Order, LineItem
└── app.py
```

Whatever you import and hand to `SDKStore.from_schema_classes(...)` *is* your schema. Everything else is just python.

## Picking an identity strategy

The first decision for any entity is how it's addressed. Three patterns cover almost everything.

**Natural single key.** Use when the domain already has a meaningful unique identifier:

```python
class Country(Entity):
    iso_code: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
```

`iso_code` is the address. `Country(iso_code="DE")` and `Country(iso_code="FR")` are different entities.

**Composite identity.** Use when an entity is naturally scoped — by tenant, region, locale — and the scope should be part of the address rather than a regular field:

```python
class User(Entity):
    user_id: str = Identity(primary_key=True)
    locale: str = Identity()
    name: str = Field(cardinality="single")
```

Now `(user_id, locale)` together identify a user. `User(user_id="u-1", locale="en")` and `User(user_id="u-1", locale="de")` are *different* entities with separate fact histories. Reach for this when the scope is a real partition of your data; don't reach for it just to add a field to the address.

**Surrogate key with a factory.** Use when there is no meaningful natural key — typically for entities that exist purely to relate other entities:

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
```

`default_factory="uuid4"` lets the SDK assign a fresh identity at write time when none is supplied. Available factories are documented in the reference; `"uuid4"` is the common one.

A useful test: if removing the field would make the entity ambiguous in the domain, it belongs in `Identity`. If two entities can sensibly differ in this field while being "the same thing" in some larger sense, it belongs in `Field`.

## Picking cardinality

Each `Field` is `cardinality="single"` or `cardinality="multi"`. The choice is about what the *snapshot* should show, not about how the ledger stores history. Both cardinalities preserve every assertion in the ledger; they differ only in what reduces out at read time.

Choose `single` when the entity has at most one current value for the field at any time:

```python
name: str = Field(cardinality="single")
home_country: Country = Field(cardinality="single")
date_of_birth: datetime = Field(cardinality="single")
```

A new assertion replaces the old one in the snapshot. The ledger keeps both; the projection shows the latest non-retracted.

Choose `multi` when the entity has a *set* of current values that coexist:

```python
tag: str = Field(cardinality="multi")
alias: str = Field(cardinality="multi")
preferred_language: Language = Field(cardinality="multi")
```

The snapshot accumulates every asserted-and-not-retracted value into a set. Multiple `add` calls with the same value collapse to one entry in the snapshot but remain distinct ledger entries — which is what makes the *count of times someone added "vip"* still answerable from the audit trail, even though the snapshot just shows `{"vip"}`.

If the field naturally holds a small fixed number of distinct values (a status, a primary contact), it's `single`. If it holds an open-ended set (tags, aliases, roles), it's `multi`.

## Modelling relationships

factpy gives you two ways to model a connection between entities. Both work; they answer different questions.

**Direct entity reference.** When a relationship is purely directional and has no attributes of its own, declare an `entity_ref` field on the source:

```python
class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    home_country: Country = Field(cardinality="single")
```

Asserting `home_country` is asserting one fact: a single predicate, with the country's address as the value. Easy to query, easy to reduce. Use this whenever the relationship has no attributes and no independent identity.

**Relationship as entity.** When the relationship has its own attributes, lifecycle, or history, model it as a separate entity:

```python
class LivesIn(Entity):
    uid: str = Identity(primary_key=True, default_factory="uuid4")
    person: Person = Field(cardinality="single")
    country: Country = Field(cardinality="single")
    since: int = Field(cardinality="single")
    until: int = Field(cardinality="single")
```

Now the relationship is a first-class subject. It can carry its own facts (`since`, `until`), accumulate its own provenance, and be the subject of its own rules. Multiple `LivesIn` instances can exist for the same person — which is exactly what you want for someone who has lived in two countries.

The decision rule is simple: *does this relationship have anything to say about itself?* If no, use a direct reference. If yes, give it an entity.

## Metadata on entities

An inner `Meta` class lets you attach metadata that travels with the entity into authoring artifacts and audit packages:

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

`version` becomes part of the schema digest, so a version bump is visible to anyone opening a ledger. `description` falls back to the docstring if not given. `tags` are free-form string labels — useful for grouping entities by domain, ownership, or compliance scope when you have many of them.

Only `version`, `description`, and `tags` are accepted in `Meta`. Anything else raises at class-definition time.

## Splitting across modules

Schemas grow. You can split them along whatever axis makes sense — domain, ownership, layer:

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

The schema is then assembled where the store is opened:

```python
from myapp.schema.identity import Person, Country
from myapp.schema.commerce import Order, LineItem

sdk = SDKStore.from_schema_classes([Person, Country, Order, LineItem])
```

There is no ceremony to splitting and no ceremony to recombining. An entity referenced as an `entity_ref` in another module just needs to be imported when the dependent module is imported; the kernel doesn't otherwise care where classes come from.

For a library or vendored schema package, expose the entity classes from a top-level `__init__.py` and let consumers compose them with their own.

## Validating before opening

Before opening a store — especially in tests or CI — `schema_preflight_from_classes` runs the same compilation that `from_schema_classes` does and returns a structured report without touching any ledger:

```python
from kernel.sdk import schema_preflight_from_classes

report = schema_preflight_from_classes([Person, Country, Order])
```

If anything is wrong — a missing `primary_key`, a circular reference, a type-domain mismatch — it surfaces as a structured error rather than a runtime exception during the first write. Run this in your test suite to catch schema regressions early.

## Opening the store

Two modes of operation. **In-memory** for tests and short-lived computation:

```python
from kernel.sdk import SDKStore

sdk = SDKStore.from_schema_classes([Person, Country, Order])
```

The ledger lives for the lifetime of the process and disappears when it ends.

**Persistent** for a long-running service or any state you want back tomorrow:

```python
sdk = SDKStore.from_schema_classes(
    [Person, Country, Order],
    ledger_path="./data/factpy.db",
)
```

The first time you open with a `ledger_path`, the schema digest is written into the file. On subsequent opens, the digest of the schema you're opening with is compared against the stored one. If they differ — which means a field, identity, or type domain has changed — the open fails with a clear schema-mismatch error rather than silently misreading old facts.

This is the single guard against schema drift on a persistent ledger. Heed it.

## Evolving a schema

What's safe to change without bumping the schema:

- Adding a new entity.
- Adding a new field to an existing entity.
- Relaxing optional metadata (descriptions, tags).
- Adding a new `Identity` field that has a `default` or `default_factory`.

What requires a new schema version, and likely a migration:

- Renaming a field or an entity.
- Changing a field's cardinality (single ↔ multi).
- Changing a field's type domain.
- Changing the identity composition of an existing entity.
- Removing a field that has been written to.

The kernel will refuse to open a ledger written under a different schema digest. Migration is an explicit step: bump the entity's `Meta.version`, write a migration that replays the old ledger into a new one under the new schema, and only then open the new ledger with the new schema.

For first-week-of-development work where the schema is still molten and there's no production ledger, just delete the file and reopen. The digest check is there to protect data that exists; nothing else.

## Where to next

- The [Reading and writing guide](reading-and-writing.md) (when written) covers the day-to-day ergonomics: `set`, `add`, `retract`, batches, attaching metadata.
- The [SDK reference](../reference/sdk.md) (when written) lists every entry point with its full signature and options.
- For background on why the schema looks the way it does, see [entities and fields](../concepts/entities-and-fields.md) and [the ledger](../concepts/the-ledger.md).