---
title: "Writing Rules"
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

# Writing rules

This guide walks through the rule DSL the way you actually use it: building up bodies from atoms, joining across entities, composing rules into larger rules, handling negation, and running them with the row format that matches your downstream code. For the conceptual picture — projections, derivations, candidates — see [rules and derivations](../concepts/rules-and-derivations.md).

## The shape of a rule

A `Rule` has an id, a version, a `select` head, and a `where` body. Variables that appear in the body are produced by the `vars(...)` context manager and projected through the head:

```python
from kernel.sdk import Rule, Pred, Not, RuleRef, vars

with vars("p", "n") as (p, n):
    name_lookup = Rule(
        id="q.name",
        version="1.0.0",
        select=[n],
        where=[Person(p), p.name == n],
    )
```

`p` and `n` are placeholders. The body says: *there exists a `Person` `p` whose `name` is `n`*. The head says: *return `n`*. Run it and you get one row per matching binding.

`id` and `version` aren't decoration. They show up in audit records, evidence trees, and `RuleRef` lookups. Treat them like the names of functions in a library — pick stable ones, version them when the body's meaning changes.

## Three body atoms

Almost every body is built from three things.

**`Pred(predicate, subject, value)`** asserts a fact must hold. Predicate names are the lowercase `entity:field` form the kernel uses internally:

```python
Pred("person:tag", p, "vip")  # p has tag "vip"
```

**Field equality on entity handles** is the typed shorthand for the same idea, scoped to a particular entity class:

```python
Person(p),
p.name == n,
p.locale == "en",
```

`Person(p)` declares that `p` ranges over `Person` entities. `p.name == n` binds `p`'s name to the variable `n`. `p.locale == "en"` filters to the literal value `"en"`. The two forms — `Pred(...)` and `entity.field == ...` — overlap; use whichever reads more clearly in context. The handle form is preferred when you're already constraining the entity type.

**`Not([...])`** says: this set of facts must *not all* hold. We'll come back to it after joins.

## Joining across entities

Rules join by *sharing a variable* between two atoms. To find people who live in Germany, share the `country` variable:

```python
with vars("p", "rel", "c") as (p, rel, c):
    germans = Rule(
        id="q.germans",
        version="1.0.0",
        select=[p],
        where=[
            Person(p),
            LivesIn(rel),
            rel.person == p,
            rel.country == c,
            Country(c),
            c.iso_code == "DE",
        ],
    )
```

The `LivesIn` relationship is a separate entity (see [defining a schema](defining-a-schema.md)), so the join walks: every `Person` `p` that appears as the `person` field of some `LivesIn` whose `country` is `c`, where `c` is the country with iso code `DE`.

For a direct `entity_ref` field — a `Person` with `home_country` declared inline — the join is shorter:

```python
with vars("p", "c") as (p, c):
    germans = Rule(
        id="q.germans_direct",
        version="1.0.0",
        select=[p],
        where=[
            Person(p),
            p.home_country == c,
            Country(c),
            c.iso_code == "DE",
        ],
    )
```

Either way, the rule is doing the same logical thing: walking the predicate graph and keeping the bindings that satisfy every clause.

## Composing rules

Once a rule names a useful concept, you can reuse it inside another rule via `RuleRef`:

```python
with vars("p") as (p,):
    vip_in_good_standing = Rule(
        id="q.vip_ok",
        version="1.0.0",
        select=[p],
        where=[
            Person(p),
            Pred("person:tag", p, "vip"),
            Not([Pred("person:tag", p, "blocked")]),
        ],
    )

with vars("p") as (p,):
    high_priority = Rule(
        id="q.high_priority",
        version="1.0.0",
        select=[p],
        where=[
            RuleRef(vip_in_good_standing)(p),
            Pred("person:tag", p, "active"),
        ],
    )
```

`RuleRef(vip_in_good_standing)(p)` injects the body of the named rule with `p` bound to the matching variable. This is composition, not duplication: change `vip_in_good_standing` and `high_priority` changes with it. Both rules' identities are preserved in any downstream evidence record, so an audit reader sees that *high priority* depended on *vip in good standing*, not on a copied-out clause.

You can also reference a rule by id and version explicitly when the rule object isn't in scope:

```python
RuleRef("q.vip_ok", version="1.0.0")(p)
```

This is the form to use across module boundaries — it doesn't require importing the rule object, just naming it.

## Negation, in practice

`Not([...])` reads naturally:

```python
Not([Pred("person:tag", p, "blocked")])
```

But its semantics is **negation as failure**: the clause succeeds when the ledger does not currently support the negated body. That is *not* the same as *"someone has asserted this is false"*. The ledger has no representation of *"`p` is not blocked"* — only the absence of an assertion that `p` *is*.

For most application code this is the right behaviour. The sharp edge appears when data arrives from multiple sources at different times: a rule that returned *Alice is not blocked* before an ingestion arrived may flip after it. The conclusion was correct given what the system knew, and the audit trail preserves the ledger state the rule ran against — but the conclusion is, by construction, defeasible.

A practical rule of thumb: when you reach for `Not`, ask whether the *absence* of an assertion really means what you want it to mean in your domain. Sometimes yes (no blocked tag → not blocked). Sometimes the right model is an explicit assertion (`status="active"`) and a positive query rather than a negation.

## Running a rule

`sdk.run(rule)` returns a list of rows shaped according to `row_format`:

```python
sdk.run(name_lookup, row_format="dict")
# [{"n": "Alice"}, {"n": "Bob"}]

sdk.run(name_lookup, row_format="tuple")
# [("Alice",), ("Bob",)]

sdk.run(germans, row_format="instance")
# [<Person snapshot ...>, <Person snapshot ...>]
```

`"dict"` is the most ergonomic for ad-hoc work — keys are the variable names from `select`. `"tuple"` is useful when you'll feed the rows to another consumer that wants positional data. `"instance"` returns entity snapshots when the head is a single entity variable, so you can read fields off them as if you'd called `sdk.get` for each.

A default row format can be set at store-open time:

```python
sdk = SDKStore.from_schema_classes([Person, Country], default_row_format="dict")
```

Per-call `row_format=` overrides the default.

## Query: a leaner shape for one-off reads

When you don't need a named, versioned, reusable rule — just a pull from the ledger — `Query` is the lighter alternative:

```python
from kernel.sdk import Query

with vars("p", "loc", "nm") as (p, loc, nm):
    q = Query(
        head=[Person(p), Person.name(locale=loc, name=nm)],
        where=[Person(p), p.locale == loc, p.name == nm],
    )

rows = sdk.run(q)
```

`Query` has the same body language as `Rule` but no id, no version, and a richer head that can shape the output directly. Use it for inline reads in application code; reach for `Rule` when the result is something you'll name, version, run again, or compose into other rules.

## Naming and versioning

Rule ids are flat strings; the convention in factpy codebases is `<kind>.<short_name>` — `q.vip_ok` for queries, `drv.auto_alias` for derivations, `m.<...>` for monitoring rules. The kernel doesn't enforce the prefix; it just helps when the rule list grows.

Version your rule when the *meaning* of its body changes — when a row that would have matched yesterday no longer does, or vice versa. Don't version it for cosmetic edits (renamed variables, reordered clauses) that don't change the result set. The version travels with every rule run into the audit trail; a version bump is how an audit reader knows an answer set today shouldn't be compared directly to last quarter's.

When two versions of a rule need to coexist — during a migration, or when multiple downstream consumers need different semantics — give them different ids. `q.vip_ok` and `q.vip_ok_v2` is clearer than relying on version-string ordering.

## Where to next

- The [Running derivations guide](running-derivations.md) (when written) covers the candidate-and-accept lifecycle for rules whose head writes new facts.
- The [Using adapters guide](using-adapters.md) (when written) covers running the same rules under PyReason, ProbLog, or Souffle.
- For the conceptual picture — what a rule is, why it has a version, why a query is a projection — see [rules and derivations](../concepts/rules-and-derivations.md).