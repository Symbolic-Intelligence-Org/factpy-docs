---
title: "02 — Rules and Derivations"
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

# Tutorial 02 — Rules and derivations

In [tutorial 01](01-sdk-basics.md) we wrote facts directly into the ledger. This tutorial covers the part of factpy that *asks questions of the ledger and proposes new facts based on the answers*: the rule DSL and the derivation lifecycle. By the end you'll have written queries, composed rules, and walked a candidate fact from proposal to acceptance with its evidence preserved.

Pick up where the last tutorial ended — same schema, same in-memory store. We'll reuse the `Person`, `Country`, and `LivesIn` declarations and add a few more facts to make the rules interesting.

## Setup

The new imports for this tutorial:

```python
from kernel.sdk import (
    SDKStore, Entity, Identity, Field,
    Rule, Query, Derivation, Pred, Not, RuleRef, vars,
)
```

`Rule`, `Query`, and `Derivation` are the three things you'll declare. `Pred`, `Not`, and `RuleRef` are body atoms. `vars` is the context manager that produces logic variables.

We'll re-declare the schema briefly and seed a few entities:

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

sdk = SDKStore.from_schema_classes(
    [Country, Person, LivesIn],
    default_row_format="dict",
)

tx = sdk.batch(meta={"source": "tutorial-02"})
de = tx.entity(Country, iso_code="DE"); de.name.set("Germany")
fr = tx.entity(Country, iso_code="FR"); fr.name.set("France")
us = tx.entity(Country, iso_code="US"); us.name.set("United States")

alice = tx.entity(Person, person_id="p-001", locale="en")
alice.name.set("Alice"); alice.tag.add("vip")

bob = tx.entity(Person, person_id="p-002", locale="en")
bob.name.set("Bob"); bob.tag.add("vip"); bob.tag.add("blocked")

carol = tx.entity(Person, person_id="p-003", locale="en")
carol.name.set("Carol"); carol.tag.add("staff")

ra = tx.entity(LivesIn); ra.person.set(alice); ra.country.set(de)
rb = tx.entity(LivesIn); rb.person.set(bob);   rb.country.set(de)
rc = tx.entity(LivesIn); rc.person.set(carol); rc.country.set(fr)

tx.commit()
```

Three people, three countries, three relationships. Alice is a VIP, Bob is a VIP but blocked, Carol is staff. Two live in Germany, one in France.

## A first query

The simplest rule asks: *whose name do we know?*

```python
with vars("p", "n") as (p, n):
    name_query = Rule(
        id="q.names",
        version="1.0.0",
        select=[n],
        where=[Person(p), p.name == n],
    )

rows = sdk.run(name_query)
print(rows)
# [{'n': 'Alice'}, {'n': 'Bob'}, {'n': 'Carol'}]
```

A few things to unpack. `vars("p", "n")` produces logic variables `p` and `n`. `Person(p)` declares that `p` ranges over `Person` entities. `p.name == n` binds `p`'s name to `n`. The head `select=[n]` says: return one row per binding, with `n` as the column.

We pass the rule to `sdk.run` and get rows back. With `default_row_format="dict"` set on the store, rows come back as dicts; you can also ask for tuples or instance snapshots per call (`row_format="tuple"`, `row_format="instance"`).

## A query that joins

To find everyone who lives in Germany, we share the `LivesIn` and `Country` variables across the body:

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

rows = sdk.run(germans, row_format="instance")
for snap in rows:
    print(snap.ref, snap.name)
# person:p-001|en  Alice
# person:p-002|en  Bob
```

Because we asked for `row_format="instance"`, each row is an entity snapshot — same shape we got from `sdk.get` in tutorial 01, complete with `.name`, `.tag`, and the `assertions` namespace. That's often the most ergonomic shape when the head is a single entity variable.

## Negation

Bob is a VIP, but he's blocked. We probably don't want him in our list of *VIPs in good standing*. That's where `Not` comes in:

```python
with vars("p") as (p,):
    vip_ok = Rule(
        id="q.vip_ok",
        version="1.0.0",
        select=[p],
        where=[
            Person(p),
            Pred("person:tag", p, "vip"),
            Not([Pred("person:tag", p, "blocked")]),
        ],
    )

rows = sdk.run(vip_ok, row_format="instance")
for snap in rows:
    print(snap.ref, snap.name)
# person:p-001|en  Alice
```

Just Alice. Bob is filtered out by the `Not` clause, Carol by the absence of the `vip` tag.

A subtle but important point: the `Not` clause says *"the ledger does not currently support that this person is blocked"*. It is **not** a claim that someone has asserted *not blocked* — there is no such concept in the ledger. The model is *negation as failure*, and it's worth thinking about what that means for your data: if a future ingestion arrives that *would* tell us Bob is also `blocked` somewhere, this rule's answer set could change. The audit trail preserves the ledger state the rule ran against, so an auditor can always reconstruct what was true at the time.

## Composing rules

Now suppose we want *high-priority VIPs*: VIPs in good standing who also have a `priority` tag (Alice doesn't, currently — but the rule should be ready). Rather than re-stating the *VIP in good standing* logic, reference the existing rule:

```python
with vars("p") as (p,):
    high_priority = Rule(
        id="q.high_priority",
        version="1.0.0",
        select=[p],
        where=[
            RuleRef(vip_ok)(p),
            Pred("person:tag", p, "priority"),
        ],
    )

print(sdk.run(high_priority))
# []  — nobody has the priority tag yet
```

`RuleRef(vip_ok)(p)` injects the body of `vip_ok` with `p` bound to the matching variable. If we change `vip_ok` later (say, to also exclude `archived` people), `high_priority` changes with it. And both rule identities — `q.vip_ok` and `q.high_priority` — show up in any downstream evidence record, so an audit reader can see *that* a high-priority match depended on the VIP-OK definition, not just on a copied-out clause.

## From rules to derivations

A rule returns rows. A **derivation** does almost the same work, but instead of returning rows, it *proposes new facts* based on the matches. Where a rule's head is a list of variables, a derivation's head is a *field call* describing what fact each match should propose.

Suppose we want to derive a tag from a person's name — for whatever reason: maybe a downstream system wants names available as tags. Here's the derivation:

```python
with vars("p", "loc", "nm") as (p, loc, nm):
    name_as_tag = Derivation(
        id="drv.name_as_tag",
        version="1.0.0",
        where=[Person(p), p.locale == loc, p.name == nm],
        head=Person.tag(person_id=p.identity("person_id"),
                        locale=loc, tag=nm),
    )
```

The body is a normal rule body. The head says: for each match, propose a `Person.tag` fact whose identity is taken from the bindings (`person_id`, `locale`) and whose value is `nm`.

(The exact identity-binding syntax can differ slightly depending on how your schema declares identities — see the [running derivations guide](../guides/running-derivations.md) for the patterns. The key point is: the head names the fact to propose, the body names the conditions under which to propose it.)

## Evaluating

Running a derivation produces *candidates*, not facts:

```python
candidates = sdk.evaluate(name_as_tag, mode="native")
print("candidates:", len(candidates))
# 3
```

Three candidates — one per person. None of them is in the ledger yet. They are proposals: *"if you accept the rule that a person's name should also be a tag, here are the facts that follow."*

Each candidate carries its evidence. You can inspect what supports a particular proposal:

```python
c = candidates[0]
print("proposed value:", c.head)
print("rule:", c.rule_id, "v", c.rule_version)
print("supporting facts:")
for support in c.evidence:
    print("  ", support)
```

The evidence names the rule, its version, and the ledger entries that produced the match — the entity itself, its locale assertion, its name assertion. That bundle is what makes the proposal auditable rather than opaque.

## Accepting

To turn a candidate into a real fact, accept it:

```python
result = sdk.accept(candidates[0],
                    approved_by="tutorial",
                    note="auto-derived")
```

That writes the candidate's head as a new assertion in the ledger. The rule id, the rule version, and the supporting ledger entries are preserved as provenance on the new fact. Anyone reading the audit story later will see *this fact was derived, by this rule, on the basis of these supports* — not just "the value showed up somewhere".

For multiple candidates at once, use `accept_many`:

```python
remaining = candidates[1:]
batch_result = sdk.accept_many(remaining, mode="atomic")
```

`mode="atomic"` writes all-or-nothing — either every accept succeeds or none of them are visible. `mode="best_effort"` writes what it can and reports failures separately.

You can preview an accept with `dry_run=True`:

```python
preview = sdk.accept(some_candidate, approved_by="tutorial", dry_run=True)
```

Useful when you want to show a reviewer "here's what would happen if you click Accept" without actually committing.

## Verifying the result

Did the accept work?

```python
snap = sdk.get(Person, person_id="p-001", locale="en")
print(snap.tag)
# {'vip', 'Alice'}  — Alice is now both 'vip' (asserted directly) and 'Alice' (derived)
```

The derived fact is in the ledger like any other. It happens to have richer provenance — the rule and its evidence — than a directly-written fact, but otherwise it's just a fact.

This is the property that distinguishes factpy's audit story from a system that *infers* facts opaquely: *every* derived fact in the ledger went through `evaluate` → review → `accept`, and the audit package reflects that explicitly. There's no third category of "facts that just appeared because the engine thought so."

## Confidence and multiple bodies

For derivations where there are multiple paths to the same conclusion with different levels of trust, factpy supports multi-body rules. We won't run one in this tutorial, but the shape is:

```python
from kernel.sdk import Body

with vars("p", "loc", "tg") as (p, loc, tg):
    vip_inference = Derivation(
        id="drv.vip",
        version="1.0.0",
        where=[
            Body([Person(p), p.locale == loc,
                  Pred("profile:vip", p, True)],
                 confidence=0.95),
            Body([Person(p), p.locale == loc,
                  Pred("model:high_value", p, tg)],
                 confidence=0.6),
        ],
        head=Person.tag(person_id=p, locale=loc, tag="vip"),
    )
```

Each `Body` is an alternative way the derivation can match. Candidates carry their body's confidence forward, and your review step can branch on it — auto-accept the high-confidence ones, route the lower-confidence ones to human review.

For *real* probabilistic reasoning (joint distributions, marginalisation), factpy can hand off to ProbLog; that's covered in [tutorial 04](04-problog-probabilistic.md).

## Recap

In this tutorial you've:

- written a query and run it three different ways (`dict`, `tuple`, `instance` row formats),
- joined across entities by sharing a variable through a relationship,
- used `Not` for negation and learned the *negation as failure* caveat,
- composed rules with `RuleRef`,
- written a derivation, evaluated it to get candidates, and accepted them as new facts,
- seen how evidence flows from candidate to ledger as provenance.

The pattern — *evaluate, review, accept* — is the heart of factpy's reasoning model. Every "thing the system concluded" that's in the ledger went through it, with a decision recorded.

## Where to next

- [Tutorial 03 — PyReason propagation](03-pyreason-propagation.md) for graph-based reasoning over annotated relationships.
- [Tutorial 04 — ProbLog probabilistic](04-problog-probabilistic.md) for actual probabilistic logic programming on uncertain facts.
- [Writing rules guide](../guides/writing-rules.md) and [running derivations guide](../guides/running-derivations.md) for the reference angle on the same material.