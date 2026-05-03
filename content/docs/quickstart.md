---
title: "Quickstart"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Quickstart

Five minutes. One example. By the end, you will have written facts to an append-only ledger, asked a logical question with negation, and traced an answer back to its source.

## What we're building

A tiny membership tracker. It knows about a few people. Some are flagged as VIPs. Some have been blocked. We'll ask one question — *"Who is a VIP in good standing?"* — and then ask factpy to explain its answer.

This is the smallest possible end-to-end factpy story: schema, writes, reads, a rule, an explanation. Every larger factpy program has these same bones.

## Install

```bash
pip install factpy-kernel
```

factpy-kernel needs Python 3.10 or newer. The only required dependencies are `pydantic` and `diskcache`.

## 1. Define the schema

We'll model one thing: a person, with a name and zero or more tags.

```python
from kernel.sdk import Entity, Identity, Field

class Person(Entity):
    person_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")
```

Two ideas to notice:

- **`Identity`** locates a fact. It's how we name a person uniquely. Every entity needs at least one primary identity.
- **`Field`** carries a value. `cardinality="single"` means one name per person; `cardinality="multi"` means a person can have many tags.

If you've used Pydantic or SQLAlchemy, this should feel familiar. The big difference comes next.

## 2. Open a store and write some facts

```python
from kernel.sdk import SDKStore

sdk = SDKStore.from_schema_classes([Person])

alice = sdk.ref(Person, person_id="p-1")
sdk.set(Person.name, alice, "Alice")
sdk.add(Person.tag, alice, "vip")

bob = sdk.ref(Person, person_id="p-2")
sdk.set(Person.name, bob, "Bob")
sdk.add(Person.tag, bob, "vip")
sdk.add(Person.tag, bob, "blocked")

carol = sdk.ref(Person, person_id="p-3")
sdk.set(Person.name, carol, "Carol")
sdk.add(Person.tag, carol, "member")
```

Three things just happened that look ordinary but aren't:

- `sdk.ref(...)` produced a *reference* to a person, not the person themselves. References decouple identity from state — you can hand a `ref` around without yet committing to which facts about it are true.
- `sdk.set` and `sdk.add` look like assignments, but they don't overwrite anything. They append entries to a ledger. Bob now has two distinct tag facts on record. If you later retract `blocked`, the original assertion still lives in history.
- Without a `ledger_path`, the store lives in memory and disappears when your process ends. To persist, pass `ledger_path="./data.db"` to `from_schema_classes`.

## 3. Read a snapshot

A snapshot is the *current* projection of all the facts about an entity:

```python
print(sdk.get(Person, person_id="p-2").name)
# Bob
print(sdk.get(Person, person_id="p-2").tag)
# ['vip', 'blocked']
```

The ledger remembers every individual fact you wrote. The snapshot is the convenient view on top — what you'd see if someone asked "what do we know about Bob right now?"

## 4. Ask a logical question

Now the part that makes factpy interesting. We want everyone tagged `vip` who is *not* also tagged `blocked`.

```python
from kernel.sdk import Rule, Pred, Not, vars

with vars("p", "n") as (p, n):
    vip_in_good_standing = Rule(
        id="q.vip_ok",
        version="1.0.0",
        select=[n],
        where=[
            Person(p),
            Pred("person:tag", p, "vip"),
            Not([Pred("person:tag", p, "blocked")]),
            p.name == n,
        ],
    )

print(sdk.run(vip_in_good_standing, row_format="dict"))
# [{'n': 'Alice'}]
```

What the rule says, line by line:

- `select=[n]` — return the variable `n` for each match.
- `Person(p)` — `p` ranges over `Person` entities.
- `Pred("person:tag", p, "vip")` — `p` must have the tag `vip`.
- `Not([Pred("person:tag", p, "blocked")])` — `p` must *not* have the tag `blocked`.
- `p.name == n` — bind that person's name to `n`.

Bob is filtered out because of his `blocked` tag. Alice qualifies. Carol doesn't have a `vip` tag, so she's never considered.

This is full first-order logic with negation as failure, evaluated directly against your fact ledger. No SQL, no ORM round-trips, no separate query layer.

## 5. Explain the answer

The ledger remembers not just *that* Alice is a VIP, but *when* the fact was asserted and where it came from. Pull it back out:

```python
# Helper to look up a predicate id from a field name (v0.1 ergonomics).
def pred_id(sdk, owner_type, field_name):
    for p in sdk.schema_ir["predicates"]:
        if p.get("owner_type") == owner_type and p.get("py_field_name") == field_name:
            return p["pred_id"]
    raise RuntimeError(f"predicate not found: {owner_type}.{field_name}")

print(sdk.explain_fact(pred_id(sdk, "Person", "tag"), alice, "vip"))
```

You'll get back a record of every assertion of this fact, including the timestamp and any metadata you attached at write time. If two sources both asserted Alice was a VIP — say a profile import and an ML model — both entries are there, with their respective provenance.

That's the audit story factpy is built around: every claim is traceable to the events that produced it. Rules and derivations carry their evidence forward, so you can ask the same question of *derived* facts later on.

## What just happened

In about thirty lines of code you've used every load-bearing piece of factpy:

- A typed schema with identities and fields.
- An append-only ledger of facts.
- A rule with negation, evaluated against the ledger.
- A first taste of the explain surface.

Every larger factpy program has the same shape. More entities, more rules, more sophisticated derivations, but the bones are these.

## Where to next

- **The mental model.** [Concepts → Overview](concepts/overview.md) explains why append-only, what derivations add on top of rules, and what an audit package contains.
- **By task.** The [Guides](guides/modeling-your-domain.md) are organized around what you're trying to do — modeling a domain, writing rules, exporting an audit run.
- **Runnable examples.** The repo's [`examples/`](https://github.com/your-org/hnsm-backend/tree/main/examples) directory has full Jupyter notebooks for derivations, ProbLog probabilistic reasoning, and audit-package round-trips.