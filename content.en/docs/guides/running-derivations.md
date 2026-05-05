---
title: "Running Derivations"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Running derivations

A derivation is a rule whose head writes new facts instead of returning rows. Running one is a three-step rhythm: **evaluate** to produce candidates, **review** to decide which to keep, **accept** to write them into the ledger with their evidence preserved. This guide covers each step in practice. For the conceptual picture, see [rules and derivations](../concepts/rules-and-derivations.md).

## Declaring a derivation

A `Derivation` looks like a `Rule` with a `head` that names a fact-shaped output instead of a list of variables:

```python
from kernel.sdk import Derivation, vars

with vars("u", "loc", "nm") as (u, loc, nm):
    auto_alias = Derivation(
        id="drv.auto_alias",
        version="1.0.0",
        where=[User(u), u.locale == loc, u.name == nm],
        head=User.tag(locale=loc, tag=nm),
    )
```

The `where` is a normal rule body. The `head` is a *field call* — `User.tag(...)` — that names what fact each match should propose. Identity coordinates (`locale=loc`) come from variables in the body; the field value (`tag=nm`) is whatever the body bound. One match in the body produces one candidate fact at the head.

A derivation can write multiple facts per match by passing a list:

```python
with vars("u", "loc", "nm", "tg") as (u, loc, nm, tg):
    multi = Derivation(
        id="drv.multi",
        version="1.0.0",
        where=[User(u), u.locale == loc, u.name == nm, u.tag == tg],
        head=[
            User.name(locale=loc, name=nm),
            User.tag(locale=loc, tag=tg),
        ],
    )
```

Each row from the body produces all of the listed head facts as candidates.

## Evaluate

`sdk.evaluate(derivation, mode=...)` runs the derivation against the current ledger and returns a list of candidates:

```python
candidates = sdk.evaluate(auto_alias, mode="native")
print("candidates:", len(candidates))
```

A candidate is not yet in the ledger. It is a proposal — the predicate, subject, and value the derivation says should be asserted, plus an evidence bundle naming the rule, its version, and the supporting ledger entries that produced the match. Until you accept it, no state has changed.

`mode="native"` is the built-in evaluator. For richer reasoning shapes — graph propagation, probabilistic inference, large-scale Datalog — pass `"pyreason"`, `"problog"`, or `"souffle"` (see [using adapters](using-adapters.md), when written). The same `Derivation` object works under each engine.

Evaluate is read-only and side-effect-free. You can run it as often as you like to inspect what *would* be proposed, without committing to anything.

## Review

Between evaluate and accept is the step where you decide which candidates should become facts. There is no fixed shape for this — it depends on what you're building.

For an automated pipeline, the review step is just a filter:

```python
to_accept = [c for c in candidates if c.confidence is None or c.confidence >= 0.8]
```

For a human-in-the-loop workflow, candidates are presented in a UI and a reviewer accepts them one by one (or in batches). For a fully-automated rule trusted by definition, `to_accept = candidates` and you skip the filter.

What matters is that *something* makes the decision and the decision is recorded. The accept step writes a decision-log entry alongside each ledger write, capturing who or what approved the candidate. That entry is what makes the audit trail interpretable later — *"every fact in the ledger is either directly written or accepted by an identifiable decision"* is the property that holds.

## Accept

`sdk.accept(candidate, approved_by=..., note=..., dry_run=...)` writes a single candidate to the ledger:

```python
result = sdk.accept(candidate, approved_by="reviewer-44", note="bulk import")
```

The `approved_by` and `note` flow into the decision log. The candidate's evidence — the rule id, the rule version, the supporting ledger entries — is preserved as provenance on the new assertion. Acceptance is idempotent: accepting the same candidate twice produces no second write, just a recorded duplicate.

For a batch of candidates, `accept_many` is more efficient and lets you control atomicity:

```python
result = sdk.accept_many(to_accept, mode="atomic")
```

`mode="atomic"` writes all-or-nothing: either every accept succeeds or none of them are visible. `mode="best_effort"` writes what it can and reports failures separately in the result. Use atomic when the candidates form a coherent unit (a derivation that classifies a user produces a name, a tier, and a flag — accept them together or not at all). Use best-effort for independent candidates where you want to make progress even if a few fail.

Both `accept` and `accept_many` accept `dry_run=True` to preview the writes without committing them — useful for a pre-flight check or a UI's "what will happen if I click Accept" view.

## Bodies and confidence

Real-world derivations often have multiple paths to the same conclusion with different levels of trust. `Body(...)` lets you name those paths explicitly and tag each with a confidence:

```python
from kernel.sdk import Body, Pred

with vars("u", "loc", "tg") as (u, loc, tg):
    vip_inference = Derivation(
        id="drv.vip",
        version="1.0.0",
        where=[
            Body([User(u), u.locale == loc, Pred("profile:vip", u, True)], confidence=0.95),
            Body([User(u), u.locale == loc, Pred("model:high_value", u, tg)], confidence=0.6),
        ],
        head=User.tag(locale=loc, tag="vip"),
    )
```

Each `Body` is an alternative way the derivation can match. Candidates produced by different bodies carry their body's confidence forward, and your review step can branch on that — auto-accept the high-confidence ones, queue the low-confidence ones for human review.

For derivations that need actual probabilistic reasoning over uncertain facts — distributions, joint probabilities, marginalisation — the ProbLog adapter takes the same `Derivation` and runs it under a probabilistic logic engine. The `Body`-and-confidence shape is the kernel's hand-off point: confidence values become probability annotations the engine can compose properly.

## Reading evidence after the fact

Once a candidate is accepted, the new assertion in the ledger carries a reference back to the rule and version that produced it, plus the supporting ledger entries. You can read that lineage off the assertion's metadata directly, or — for a structured view across many derived facts — load the audit package the run produced and walk the evidence trees with `kernel.audit`.

```python
from kernel.audit import load_audit_package

pkg = load_audit_package("./audits/run-2024-04-01")
for tree_id, tree in pkg.provenance_trees.items():
    # each tree shows a derived fact and its supporting entries
    ...
```

See [audit and provenance](../concepts/audit-and-provenance.md) and (when written) the [auditing a run guide](auditing-a-run.md) for how to turn a finished run into a shareable package and read it back.

## Patterns and pitfalls

**Re-running a derivation is safe.** Evaluate produces candidates from the current ledger; accept is idempotent. If a candidate has already been accepted, accepting it again is a no-op (and recorded as a duplicate in the decision log).

**Bodies are unioned, not intersected.** A multi-body derivation matches if *any* body matches. If you want every body's conditions to hold simultaneously, that's a single body with all the clauses; multi-body is for *alternative paths to the same conclusion*.

**Versioning a derivation matters.** When a derivation's body changes meaning, bump the version. The new version produces new candidates; the old version's accepted facts stay in the ledger with their original provenance, traceable back to the version of the rule that produced them. Audit readers see both, and see why both exist.

**Don't auto-accept everything by default.** The candidate-and-accept rhythm exists so that *somebody decided*. Even when the deciding is automated, the decision log entry is what makes the audit story coherent. A pipeline that calls `evaluate` and `accept_many` with no filter is fine — the decision is *"every candidate from this trusted derivation, by policy"* — but it should be that decision deliberately, not by accident.

## Where to next

- The [Auditing a run guide](auditing-a-run.md) (when written) covers exporting the package a derivation run produces and reading it back with `kernel.audit`.
- The [Using adapters guide](using-adapters.md) (when written) covers running the same derivations under PyReason, ProbLog, or Souffle.
- For the conceptual model — what a candidate is, why evidence travels with it, why acceptance is a separate step — see [rules and derivations](../concepts/rules-and-derivations.md).