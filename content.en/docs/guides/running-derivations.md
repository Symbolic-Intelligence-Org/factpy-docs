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

A derivation is a rule whose head proposes new facts when the body matches, rather than returning rows. The lifecycle of a derivation falls into three steps: evaluation produces candidates from the current ledger state, review decides which of those candidates ought to become facts, and acceptance writes the accepted candidates as new ledger entries with their evidence preserved as provenance. The present guide develops each step in practice. For the conceptual material on which the lifecycle rests, see [rules and derivations](/docs/concepts/rules-and-derivations).

## Declaring a derivation

A `Derivation` shares the body language of a `Rule` and the same identity-and-version surface, differing only in the head: where a `Rule` projects its variable bindings as rows, a `Derivation` declares a fact-shaped head that names what should be proposed for each binding the body satisfies.

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

The `where` is an ordinary rule body. The `head` is a *field call* — `User.tag(...)` — that names what fact each match should propose. Identity coordinates such as `locale=loc` are read from the body's variable bindings; the field value, here `tag=nm`, is whatever the body has bound. One match in the body produces one candidate fact at the head.

A derivation can write multiple related facts per match by passing a list of head field calls.

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

Each binding the body satisfies produces all of the listed head facts as candidates, which is the appropriate shape when several related facts together form a coherent unit that should be evaluated, reviewed, and accepted in tandem rather than separately.

## Evaluate

`sdk.evaluate(derivation, mode=...)` runs the derivation against the current ledger and returns a list of candidates.

```python
candidates = sdk.evaluate(auto_alias, mode="native")
print("candidates:", len(candidates))
```

A candidate is not yet part of the ledger. It is a proposal: the predicate, subject, and value the derivation says should be asserted, together with an evidence bundle naming the rule, its version, and the ledger entries that supported the match. Until the candidate is accepted, no state has changed in the ledger, and the evaluation can be repeated against the same ledger to inspect what would be proposed without committing to anything.

The `mode="native"` argument selects the built-in evaluator, which is the appropriate choice for ordinary logical derivations. For reasoning shapes the native evaluator does not address — graph propagation, probabilistic inference, large-scale Datalog — the alternatives are `"pyreason"`, `"problog"`, and `"souffle"`, developed in [using adapters](/docs/guides/using-adapters). The same `Derivation` object runs under each engine; what changes is the evaluator that consumes it.

Evaluation is read-only and side-effect-free, which is what permits it to be run as often as needed for inspection or for a pre-acceptance review without affecting any subsequent acceptance.

## Review

Between evaluation and acceptance is the step at which a decision is made about which candidates ought to become facts. The kernel does not prescribe a shape for this step, since the appropriate shape depends on what is being built, but in every case the decision is recorded alongside the resulting writes and is what makes the audit story interpretable later.

For an automated pipeline, the review step is typically a filter expressed in ordinary code.

```python
to_accept = [c for c in candidates if c.confidence is None or c.confidence >= 0.8]
```

For a human-in-the-loop workflow, candidates are presented in an interface and a reviewer accepts them individually or in batches. For a derivation that is trusted by policy and has no per-candidate review, the filter reduces to the identity, with `to_accept = candidates`.

The form of the decision matters less than that the decision is made and recorded. Acceptance writes a decision-log entry alongside each ledger write, capturing the identity of the reviewer or the policy that approved the candidate. The property that holds — that every fact in the ledger is either directly written or accepted by an identifiable decision — is what makes the audit trail interpretable when the run is later reviewed by a third party, and the decision-log entry is the connective tissue between the candidate and the writes it produced.

## Accept

`sdk.accept(candidate, approved_by=..., note=..., dry_run=...)` writes a single candidate to the ledger.

```python
result = sdk.accept(candidate, approved_by="reviewer-44", note="bulk import")
```

The `approved_by` and `note` arguments flow into the decision log. The candidate's evidence — the rule id, the rule version, and the supporting ledger entries — is preserved as provenance on the new assertion. Acceptance is idempotent: accepting the same candidate a second time produces no further write and is recorded as a duplicate in the decision log.

For a batch of candidates, `accept_many` is more efficient and exposes a parameter for controlling the atomicity of the batch.

```python
result = sdk.accept_many(to_accept, mode="atomic")
```

The `mode="atomic"` setting writes all-or-nothing: either every acceptance in the batch succeeds and the writes become visible together, or none of them do. The `mode="best_effort"` setting writes what it can and returns a result that reports the failures separately. The atomic mode is appropriate where the candidates form a coherent unit — a derivation that classifies a user might propose a name, a tier, and a flag together, with all three being meaningful only as a set — and the best-effort mode is appropriate for independent candidates where partial progress is preferable to none.

Both `accept` and `accept_many` accept a `dry_run=True` argument that previews the writes without committing them. The previewed result has the same shape as the real one and is the basis for a pre-flight check or for an interface that shows a reviewer what an acceptance would do before the click that performs it.

## Bodies and confidence

A derivation may have more than one path to the same conclusion, with different levels of trust attached to each path. The `Body(...)` wrapper lets these paths be named explicitly and tagged with a confidence value.

```python
from kernel.sdk import Body, Pred

with vars("u", "loc", "tg") as (u, loc, tg):
    vip_inference = Derivation(
        id="drv.vip",
        version="1.0.0",
        where=[
            Body([User(u), u.locale == loc, Pred("profile:vip", u, True)],
                 confidence=0.95),
            Body([User(u), u.locale == loc, Pred("model:high_value", u, tg)],
                 confidence=0.6),
        ],
        head=User.tag(locale=loc, tag="vip"),
    )
```

Each `Body` is an alternative match path, and the bodies are or-joined: a candidate is produced for any binding that satisfies any of them. Candidates carry the confidence of the body that produced them as opaque metadata, which a review step can branch on — auto-accepting the high-confidence candidates and routing the low-confidence ones to a human reviewer is the standard pattern.

Where confidence is meant in the strict probabilistic sense and the engine is expected to combine confidences correctly under joint distributions and marginalisation, factpy hands off to ProbLog through the optional adapter layer. The `Body`-and-confidence shape is the kernel's handoff point: confidence values that travel as opaque numbers under the native evaluator become probability annotations that ProbLog interprets and composes. The same `Derivation` object runs under either evaluator; what changes is the semantics of the confidence value.

## Reading evidence after the fact

Once a candidate has been accepted, the resulting assertion in the ledger carries a reference to the rule and version that produced it together with the supporting ledger entries. The lineage can be read off the assertion's metadata directly, or, for a structured view across many derived facts at once, an audit package produced from the run can be opened with `kernel.audit` and the evidence trees walked.

```python
from kernel.audit import load_audit_package

pkg = load_audit_package("./audits/run-2024-04-01")
for tree_id, tree in pkg.provenance_trees.items():
    # each tree shows a derived fact and its supporting entries
    ...
```

See [audit and provenance](/docs/concepts/audit-and-provenance) for the conceptual material on what is recorded during a run, and the [auditing a run guide](/docs/guides/auditing-a-run) for the practical mechanics of producing and consuming an audit package.

## Patterns and pitfalls

A few practices recur across factpy codebases and are worth naming explicitly.

Re-running a derivation is safe. Evaluation produces candidates from the current ledger without affecting it; acceptance is idempotent in the sense that a candidate already accepted will not produce a second write, and the duplicate is recorded in the decision log. There is no operational risk in evaluating the same derivation repeatedly, whether to inspect what would be proposed under different ledger states or to recover a candidate set after an interrupted review.

Bodies in a multi-body derivation are or-joined, not and-joined. A candidate is produced when any of the bodies matches, not only when all of them match. The shape for an *and*-joined condition is a single body containing all the clauses; the multi-body shape exists specifically to express *alternative paths to the same conclusion* with potentially different confidences. Confusing the two is the most common modelling mistake when first reaching for `Body(...)`.

Versioning a derivation matters in the same way that versioning a rule matters, and for the same reason. When a derivation's body changes meaning, the version should be bumped. The new version produces new candidates; the previous version's accepted facts remain in the ledger with their original provenance, traceable back to the version of the rule that produced them. The audit reader sees both versions as distinct origins for distinct facts and is therefore not misled into comparing results obtained under one version directly to those obtained under another.

Auto-acceptance is appropriate as a deliberate policy and inappropriate as a default. The candidate-and-acceptance lifecycle exists in order that some identifiable decision lies behind every fact written into the ledger; even when the decision is automated, the decision-log entry remains the connective tissue that makes the audit story coherent. A pipeline that calls `sdk.evaluate` and `sdk.accept_many` with no filter is well-formed when the decision being expressed is *every candidate from this trusted derivation, by policy* — but the policy must be the deliberate choice, not the consequence of having omitted a filter.

## Where to next

The [auditing a run guide](/docs/guides/auditing-a-run) covers exporting the package a derivation run produces and reading it back with `kernel.audit`. The [using adapters guide](/docs/guides/using-adapters) covers running the same derivations under PyReason, ProbLog, or Souffle. For the conceptual model — what a candidate is, why evidence travels with it, why acceptance is a separate step — see [rules and derivations](/docs/concepts/rules-and-derivations).