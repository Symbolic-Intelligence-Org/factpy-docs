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

This guide develops the rule DSL the way it is used in practice: building bodies from atoms, joining across entities through shared variables, composing rules into larger rules, working with negation, and running rules with the row format appropriate to the calling code. For the conceptual material — projections, derivations, candidates — see [rules and derivations](/docs/concepts/rules-and-derivations).

## The shape of a rule

A `Rule` is constructed with four structurally distinguished parts: an id, a version, a `select` head that names what the rule returns for each match, and a `where` body that lists the conditions under which a match is produced. Variables that appear in the body are produced by the `vars(...)` context manager and projected through the head.

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

The variables `p` and `n` are placeholders bound by the body and projected by the head. The body asserts that there exists a `Person` `p` whose `name` is `n`; the head specifies that `n` should be returned for each binding. Running the rule yields one row per binding the body satisfies.

The `id` and `version` are not decorative. They are the identifiers under which the kernel registers the rule and refers to it in audit records, evidence trees, and references from one rule to another. They are best treated as the names of functions in a library — chosen for stability, and bumped in version when the meaning of the body changes.

## Three body atoms

Most rule bodies are built from three kinds of atom together with the entity-class declarations that scope variables to particular schemas.

The atom `Pred(predicate, subject, value)` requires that the ledger contain an active assertion of the named predicate, with the given subject and value. Predicate names are the lowercase `entity:field` form the kernel uses internally.

```python
Pred("person:tag", p, "vip")  # p has tag "vip"
```

The typed shorthand for the same kind of constraint is *field equality on entity handles*, available wherever a variable has been scoped to a particular entity class. The class declaration `Person(p)` introduces `p` as a variable ranging over `Person` entities, and field references on that variable participate in the body as constraints.

```python
Person(p),
p.name == n,
p.locale == "en",
```

`p.name == n` binds the variable `n` to `p`'s name; `p.locale == "en"` constrains `p`'s locale to the literal `"en"`. The two forms — `Pred(...)` and `entity.field == ...` — express the same kind of condition, and the choice between them is a question of readability rather than of capability. The handle form is preferable wherever the entity's class is already constrained, since it is shorter and exposes the field types to local typing checks.

The atom `Not([...])` requires that a list of clauses *not* all hold simultaneously, under the semantics of negation as failure. It is the third atom form, and the section *Negation in practice* below develops its semantics in detail.

## Joining across entities

A rule joins across entities by sharing a variable between two atoms. To find every person who lives in Germany, expressed as a join over the `LivesIn` relationship and the `Country`, the body shares `p` between the `Person` and the `LivesIn`, and `c` between the `LivesIn` and the `Country`.

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

The `LivesIn` relationship is a separate entity in the schema (see [defining a schema](/docs/guides/defining-a-schema)), and the join walks: every `Person` `p` that appears as the `person` field of some `LivesIn` whose `country` is `c`, where `c` is the `Country` with iso code `DE`.

When the relationship is modelled instead as a direct `entity_ref` field on the source — a `Person` declared with a `home_country: Country` field inline — the join is shorter, since the reference is already part of the source entity's facts.

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

In either form, the rule is doing the same logical work: walking the predicate graph and retaining the bindings that satisfy every clause of the body.

## Composing rules

Once a rule names a useful concept, that concept can be referenced from inside another rule's body via `RuleRef`, which injects the named rule's body into the calling rule's body with the variables bound through to the call site.

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

`RuleRef(vip_in_good_standing)(p)` injects the *vip in good standing* condition as a sub-clause of the *high priority* rule's body, with `p` flowing through as the matched person. This is composition rather than duplication: a single definition of *vip in good standing* exists, and any change to it propagates to every rule that references it; both rules' identities are preserved in downstream evidence records, so an audit reader can trace a *high priority* match back through the *vip in good standing* condition that licensed it.

The cross-module form references a rule by id and version explicitly, without requiring the rule object itself to be in scope at the reference site.

```python
RuleRef("q.vip_ok", version="1.0.0")(p)
```

This is the idiomatic shape for references across module boundaries, and it is what allows a vocabulary of named rules to be assembled from multiple modules without forcing the corresponding python imports.

## Negation in practice

The atom `Not([...])` reads naturally — *not blocked*, in the example above — but its semantics is precise and worth understanding. A `Not` clause succeeds when the ledger does not currently support its body and fails when the ledger does, under what is called *negation as failure*. There is no representation in the ledger of *not blocked* as a positive claim; what the rule tests is the *absence* of an assertion that the subject is blocked, not the *presence* of an assertion that the subject is unblocked.

For most application code this is the right semantics, but the distinction has practical consequences when data arrives over time from multiple sources. A rule that returned *Alice is not blocked* before some ingestion arrived may return the opposite afterwards, with no change to the rule itself: the conclusion was correct against the ledger the rule ran against, but the ledger has since changed. The audit trail preserves the ledger state of each rule run, so the change in conclusion remains visible as a change in inputs rather than an inconsistency between two runs, but the conclusions of `Not`-clauses are by construction defeasible in a way that conclusions of positive clauses are not.

The practical guidance is that whenever a `Not` is reached for, the modeller should consider whether the *absence* of an assertion is really what is meant in the domain. Sometimes the answer is yes — no `blocked` tag legitimately means not blocked. Sometimes the absence is too weak a basis for the conclusion, and the appropriate model is an explicit positive assertion (`status == "active"`) that some upstream actor is responsible for setting and that the rule queries positively rather than negating its complement.

## Running a rule

`sdk.run(rule)` returns a list of rows shaped according to the `row_format` argument.

```python
sdk.run(name_lookup, row_format="dict")
# [{"n": "Alice"}, {"n": "Bob"}]

sdk.run(name_lookup, row_format="tuple")
# [("Alice",), ("Bob",)]

sdk.run(germans, row_format="instance")
# [<Person snapshot ...>, <Person snapshot ...>]
```

The `"dict"` format is the most ergonomic for ad-hoc work, since the keys are the variable names from `select` and the rows can be read fluently by name. The `"tuple"` format omits the names and returns positional rows, which is appropriate when the rows are passed to a downstream consumer that expects positional data. The `"instance"` format is available where the head is a single entity-typed variable, in which case each row is an entity snapshot of the same shape that `sdk.get` would produce, with field access available directly on the row.

A default `row_format` can be set at the moment the store is opened and applies to every subsequent `sdk.run` call that does not specify one.

```python
sdk = SDKStore.from_schema_classes([Person, Country], default_row_format="dict")
```

Per-call `row_format=` overrides the store-level default.

## Query: a leaner shape for one-off reads

`Query` is the lighter alternative for inline reads, used wherever a result is wanted but no named, versioned, reusable rule definition is needed.

```python
from kernel.sdk import Query

with vars("p", "loc", "nm") as (p, loc, nm):
    q = Query(
        head=[Person(p), Person.name(locale=loc, name=nm)],
        where=[Person(p), p.locale == loc, p.name == nm],
    )

rows = sdk.run(q)
```

`Query` has the same body language as `Rule` but no id, no version, and a richer head that can shape the output directly. It is the appropriate shape for reads embedded in application code; `Rule` is the appropriate shape wherever the result is something to be named, versioned, run again, or composed into further rules.

## Naming and versioning

Rule ids are flat strings, and the convention across factpy codebases is `<kind>.<short_name>` — `q.vip_ok` for queries, `drv.auto_alias` for derivations, `m.<...>` for monitoring rules. The kernel does not enforce the prefix, but the convention pays off as a rule list grows by making the kind of every rule legible from its name.

A rule's version should be bumped when the *meaning* of its body changes — that is, when a binding that would have matched yesterday no longer does, or when a binding that would not have matched yesterday now does. Cosmetic edits that leave the result set unchanged — renamed variables, reordered clauses — do not warrant a version bump. The version travels with every rule run into the audit trail, and a deliberate bump is the signal by which an audit reader knows that today's result set should not be compared directly to last quarter's.

Where two versions of a rule need to coexist — during a migration, or when multiple downstream consumers need different semantics — they are best given different ids rather than relying on version-string ordering. `q.vip_ok` and `q.vip_ok_v2` is clearer at the call site and clearer in audit records than two versions of the same id.

## Where to next

The [running derivations guide](/docs/guides/running-derivations) covers the candidate-and-accept lifecycle for rules whose head proposes new facts. The [using adapters guide](/docs/guides/using-adapters) covers running the same rules under PyReason, ProbLog, or Souffle. For the conceptual picture — what a rule is, why it carries a version, why a query is a projection — see [rules and derivations](/docs/concepts/rules-and-derivations).