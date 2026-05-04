---
title: "05 — Onboarding Journey"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Tutorial 05 — Onboarding journey

This tutorial is the capstone. We weave every move from tutorials 01–04 into a single end-to-end scenario: define a schema, write facts, query the ledger, derive new facts, accept them, export the run as an audit package, and read the package back. By the end you'll have a complete factpy workflow you can adapt to a real project.

The scenario: a small system that onboards new users into an organisation. We want to track users, their initial roles, and which of them get promoted to *audited* status by an automated rule. The rule's output should be reviewable — *why* did this user get audited? — through the audit package.

We won't introduce anything that wasn't covered in earlier tutorials. The goal here is to put it all together.

## Setup

```python
from pathlib import Path
import tempfile

from kernel.sdk import (
    SDKStore, Entity, Identity, Field,
    Derivation, Query, Pred, vars,
)
from kernel.adapters.souffle.package import ExportOptions

from kernel.audit import (
    AuditQuery,
    load_audit_package,
    build_candidate_evidence_tree_dto,
    build_candidate_evidence_tree_summary_dto,
    build_candidate_evidence_tree_narrative_dto,
)
```

Three blocks. The SDK and DSL we've been using throughout. The `ExportOptions` to declare an audit package. The `kernel.audit` reader to walk the package after export.

## The schema

A `Country` (anchor for users), and a `User` with an admin tag, a home country, and identity coordinates we've seen before:

```python
class Country(Entity):
    code: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")

class User(Entity):
    user_id: str = Identity(primary_key=True)
    locale: str = Identity()
    name: str = Field(cardinality="single")
    tag: str = Field(cardinality="multi")
    home: Country = Field(cardinality="single")

sdk = SDKStore.from_schema_classes([Country, User])
```

`User` has a composite identity (`user_id` + `locale`), a multi-valued tag set, and a single-valued reference to a `Country`. We'll use the `tag` field for both directly-asserted and derived tags.

## Writing the initial facts

Two countries, three users with various tags:

```python
de    = sdk.ref(Country, code="DE")
fr    = sdk.ref(Country, code="FR")
alice = sdk.ref(User, user_id="u1", locale="en")
bob   = sdk.ref(User, user_id="u2", locale="en")
carol = sdk.ref(User, user_id="u3", locale="en")

# Countries
sdk.set(Country.name, de, "Germany",
        meta={"source": "import", "trace_id": "tut-05"})
sdk.set(Country.name, fr, "France",
        meta={"source": "import", "trace_id": "tut-05"})

# Users
sdk.set(User.name, alice, "Alice",
        meta={"source": "import", "trace_id": "tut-05"})
sdk.set(User.home, alice, de,
        meta={"source": "import", "trace_id": "tut-05"})
sdk.add(User.tag, alice, "admin",
        meta={"source": "policy", "trace_id": "tut-05"})

sdk.set(User.name, bob, "Bob",
        meta={"source": "import", "trace_id": "tut-05"})
sdk.set(User.home, bob, fr,
        meta={"source": "import", "trace_id": "tut-05"})
sdk.add(User.tag, bob, "admin",
        meta={"source": "policy", "trace_id": "tut-05"})

sdk.set(User.name, carol, "Carol",
        meta={"source": "import", "trace_id": "tut-05"})
sdk.set(User.home, carol, de,
        meta={"source": "import", "trace_id": "tut-05"})
sdk.add(User.tag, carol, "guest",
        meta={"source": "policy", "trace_id": "tut-05"})
```

Three users: Alice and Bob are admins, Carol is a guest. We'll use the direct API here (`sdk.set` / `sdk.add` with `sdk.ref` references) to show that pattern; the batch API from tutorial 01 would work just as well.

## Reading what we wrote

Sanity-check by snapshot:

```python
snap = sdk.get(User, user_id="u1", locale="en")
print(snap.name, sorted(snap.tag), snap.home.code)
# Alice ['admin'] DE
```

Snapshots reduce the ledger under the schema's cardinality rules. Single-valued fields read as values; multi-valued fields read as sets; entity-reference fields read as nested snapshots so you can navigate (`snap.home.code`).

## Querying with the DSL

A more general read uses a `Query`. Find every user, returning their ref and name:

```python
with vars("u", "name") as (u, name):
    q = Query(
        head=[User(u), User.name(value=name)],
        where=[User(u), u.name == name],
    )

rows = sdk.run(q)
for row in rows:
    print(row)
```

`Query` is the lighter-weight cousin of `Rule`: same body language, no id, no version, with a head that can shape rows directly. Use it for inline reads; reach for `Rule` when you'll name and version the result.

## Deriving the *audited* tag

The policy: every user who carries an `admin` tag should also carry an `audited` tag. We could write the audited tags by hand, but that's exactly the kind of derived fact a derivation should produce — so we can audit *why*:

```python
with vars("u", "loc", "derived") as (u, loc, derived):
    audit_drv = Derivation(
        id="journey.derived_tag",
        version="1.0.0",
        where=[
            User(u),
            u.locale == loc,
            Pred("user:tag", u, "admin"),
            derived == "audited",
        ],
        head=User.tag(user_id=u.identity("user_id"),
                      locale=loc, tag=derived),
    )

candidates = sdk.evaluate(audit_drv, mode="native")
print(f"candidates: {len(candidates)}")
# 2 — Alice and Bob, since both have admin
```

Two candidates, one per admin user. They are not yet in the ledger. Each candidate carries:

- the proposed fact (an `audited` tag for that user),
- the rule that produced it (`journey.derived_tag` v1.0.0),
- the supporting ledger entries (the user, the locale, the admin tag).

Accept them:

```python
for cand in candidates:
    sdk.accept(cand, approved_by="onboarding-policy", note="auto-audit")
```

Re-read:

```python
print(sorted(sdk.get(User, user_id="u1", locale="en").tag))
# ['admin', 'audited']
print(sorted(sdk.get(User, user_id="u3", locale="en").tag))
# ['guest']
```

Alice now has both `admin` (directly asserted) and `audited` (derived). Carol is unchanged. The derived facts are distinguishable from the directly-asserted ones because the *provenance* on each tells a different story — but the snapshot doesn't care; both are facts about the user.

## Verifying via a query

If we want to find the *audited* users specifically, that's just a query:

```python
with vars("u") as (u,):
    audited_q = Query(
        head=User(u),
        where=[User(u), u.tag == "audited"],
    )

rows = sdk.run(audited_q, row_format="instance")
for snap in rows:
    print(snap.ref, snap.name, sorted(snap.tag))
# user:u1|en  Alice  ['admin', 'audited']
# user:u2|en  Bob    ['admin', 'audited']
```

Two users, both correctly audited. The query doesn't know which tags were directly asserted and which were derived — that distinction belongs to provenance, not to the snapshot.

## Exporting an audit package

So far the run has happened entirely in memory. To make it reviewable elsewhere — by a colleague, in CI, by a compliance reader — we export an audit package:

```python
package_dir = Path(tempfile.mkdtemp(prefix="factpy_journey_"))
sdk.export_package(package_dir, ExportOptions(package_kind="audit"))
print(f"package at: {package_dir}")
```

That writes a self-contained directory: schema, rules, run records, candidate ledger, accept-write ledger, decision log, evidence trees. The full layout is covered in [auditing a run](../guides/auditing-a-run.md); for the tutorial, the relevant point is that *one call gives you a complete review-ready bundle*.

## Reading the package back

`kernel.audit` reads packages without depending on the rest of factpy. Load the package and wrap it in an `AuditQuery` for ergonomic access:

```python
package = load_audit_package(package_dir)
audit = AuditQuery(package)

runs = audit.list_runs()
accepted = audit.list_accepted_candidates()

print(f"runs in package: {len(runs)}")
print(f"accepted candidates: {len(accepted)}")
```

`AuditQuery` is the navigation surface — it knows how the package is structured and gives you typed accessors for runs, candidates, decisions, and evidence. You don't have to walk JSON files by hand.

## Inspecting a candidate's evidence

Pick one of the accepted candidates and walk its evidence tree:

```python
candidate_id = accepted[0].candidate_id

raw_tree    = build_candidate_evidence_tree_dto(audit, candidate_id)
summary     = build_candidate_evidence_tree_summary_dto(audit, candidate_id)
narrative   = build_candidate_evidence_tree_narrative_dto(audit, candidate_id)

print("=== summary ===")
print(summary.summary_text)
print("\n=== narrative ===")
print(narrative.narrative_text)
```

Three views of the same evidence:

- **Raw tree** — the full structure: candidate at the root, each body match below it, supporting ledger entries at the leaves.
- **Summary** — a compact textual rendering for review UIs and dashboards.
- **Narrative** — a human-readable English-prose explanation of why the candidate was produced.

For Alice's `audited` candidate, the narrative will say something like: *"`audited` was proposed for `User(u1, en)` by rule `journey.derived_tag` v1.0.0, supported by the assertion of `User(u1, en)`'s `admin` tag at entry #5, and the resolution of `u1`'s locale to `en` at entry #2."* The exact wording is engine-determined; the structure — *what got proposed, by what rule, supported by what facts* — is what reviewers see.

This is the property the audit story is built on: *every derived fact in the ledger has an evidence trail, and that trail is mechanically inspectable*. It's not a log file; it's a structured graph linking conclusions back to the leaf facts that supported them, with the rule that did the linking named explicitly.

## What we built

A complete factpy workflow:

1. A typed schema with two entities, an entity-reference field, and a multi-valued tag.
2. Direct fact-writes with provenance metadata.
3. Snapshot reads and a `Query` over the ledger.
4. A `Derivation` that proposes new facts based on existing ones.
5. An `accept` step that turns candidates into ledger writes with evidence preserved.
6. An audit package export, capturing the entire run as a self-contained directory.
7. An audit-package read, walking from the high-level run down to a specific candidate's evidence tree.

This is the shape every factpy project ends up having. Different schemas, different rules, possibly different engines (PyReason or ProbLog instead of the native evaluator) — but the same structural arc: write facts, ask questions, derive conclusions deliberately, keep the trail.

## Where to next

The tutorials are done. From here:

- The [guides](../guides/_index.md) cover each move in task-oriented depth — pick the one that matches what you're stuck on.
- The [concepts](../concepts/_index.md) cover the conceptual model when you want to step back from the *how* and understand the *why*.
- The [reference](../reference/_index.md) (when written) covers the full API surface — every entry point, every option, every error code.

Build something with it.