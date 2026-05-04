---
title: "04 — ProbLog Probabilistic"
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

# Tutorial 04 — ProbLog probabilistic reasoning

The native engine reasons about facts as either-true-or-not-asserted. The `Body(..., confidence=...)` shape from [tutorial 02](02-rules-and-derivations.md) lets you tag whole bodies with a confidence — useful for review-time filtering, but not the same as actually *reasoning probabilistically*. When you need real probabilistic semantics — facts that are true with some probability, rules that combine probabilities properly, queries that return marginal probabilities — you reach for ProbLog.

This tutorial walks through a small probabilistic-reasoning scenario: inferring researcher expertise from noisy signals about their work. Each signal is a probabilistic observation; the rule combining them produces probabilities, not certainties. We'll write the derivation, run it under ProbLog, and accept the resulting probabilistic facts back into the ledger.

Reach for this tutorial when:

- your input data has *probabilities*, not just confidence labels,
- you need rules to combine probabilities in a principled way (not just take the max or average),
- you want to ask "what is `P(X)` given everything we know?" rather than "is `X` derivable?",
- the `Body(..., confidence=...)` shape feels too coarse for your problem.

If you only need *"some confidence value travels with the answer"*, the native engine with weighted bodies is enough.

## Setup

ProbLog ships as an optional dependency:

```
pip install factpy-kernel[problog]
```

ProbLog also requires the `problog` Python package and (under the hood) a working install of its solver. The `factpy-kernel[problog]` extra pulls these in.

The imports for this tutorial:

```python
from kernel.sdk import SDKStore, Entity, Identity, Field
from kernel.sdk.dsl import Derivation, Pred, vars as sdk_vars

import kernel.adapters.problog                      # registers the engine
from kernel.adapters.problog.rule_ext import ProbLogRuleExt
from kernel.adapters.problog.accept import persist_problog_annotations
```

The `import kernel.adapters.problog` line is doing real work: it registers the ProbLog evaluator with `sdk.evaluate`, so subsequent `mode="problog"` calls find an implementation. Without that import, evaluating a ProbLog derivation raises a *no engine registered* error.

## A minimal probabilistic schema

A `Researcher` with a few signal-shaped fields:

```python
class Researcher(Entity):
    researcher_id: str = Identity(primary_key=True)
    name: str = Field(cardinality="single")
    expertise: str = Field(cardinality="single")
    impact_score: str = Field(cardinality="single")

sdk = SDKStore.from_schema_classes([Researcher])
```

Nothing schema-specific to ProbLog so far. The probability information enters through write-time metadata.

## Writing facts with probabilities

Each assertion gets a `confidence` in its metadata, which ProbLog will read as the probability of the observation:

```python
tx = sdk.batch(meta={"source": "tutorial-04"})

r1 = tx.entity(Researcher, researcher_id="r-001")
r1.name.set("Alice")
r1.impact_score.set("high",
                    meta={"confidence": 0.85, "source": "citations"})

r2 = tx.entity(Researcher, researcher_id="r-002")
r2.name.set("Bob")
r2.impact_score.set("medium",
                    meta={"confidence": 0.62, "source": "citations"})

tx.commit()
```

`confidence: 0.85` says: *we observed Alice's impact score is "high", and we are 85% sure of that observation*. ProbLog will treat this as the input probability when reasoning.

This is the same `confidence` key you'd use in any other factpy code; ProbLog just *also* knows to read it. Code that writes facts doesn't have to care that something downstream might reason about them probabilistically.

## A ProbLog derivation

The derivation looks like a regular `Derivation` with two extras: `mode="problog"` and an `engine_ext` that carries probabilistic configuration:

```python
with sdk_vars("r", "name") as (r, name):
    expertise_drv = Derivation(
        id="drv.expertise_discovery",
        version="v1",
        where=[Pred("researcher:name", r, name)],
        target="researcher:expertise",
        head_vars=[r, name],
        mode="problog",
        engine_ext=ProbLogRuleExt(branch_probabilities=(0.85,)),
    )
```

A few things to unpack.

`mode="problog"` tells `sdk.evaluate` which engine to dispatch to.

`engine_ext=ProbLogRuleExt(...)` carries engine-specific configuration. `branch_probabilities=(0.85,)` is the probability of the rule's disjunctive branch — *how often, given that the body matches, should the head be considered to follow?* A regular logical rule has an implicit branch probability of 1.0; lowering it expresses *"this rule fires with some probability, not always"*.

`target=` and `head_vars=` are the ProbLog-specific way to declare the head (the predicate it concludes about, and the variables that flow into it). For derivations that only run under ProbLog, this is the natural shape. Native-mode derivations use the field-call head shown in tutorial 02.

The body — `where=[Pred("researcher:name", r, name)]` — is a regular DSL body. Same atoms, same join semantics. The engine reading it differs.

## Evaluating

`sdk.evaluate` dispatches to the registered ProbLog evaluator:

```python
candidates = sdk.evaluate(
    expertise_drv,
    engine_options={"timeout": 10},
)
print(f"ProbLog candidates: {len(candidates)}")
```

Behind the scenes, the adapter:

1. exports the relevant ledger facts and the rule into ProbLog program text,
2. runs ProbLog (with `--trace` enabled, so we get reasoning traces),
3. parses the output probabilities and proofs,
4. returns a `CandidateSet` with each candidate carrying its probability and provenance.

`engine_options` is for engine-specific tuning — timeouts, solver flags, search depth limits. The exact keys are documented in the [adapters reference](../reference/adapters.md) (when written).

You can inspect what came back:

```python
for cand in candidates:
    print(f"  {cand.head}: P={cand.confidence:.3f}")
```

Each candidate's `confidence` is now a *probability*, not just an arbitrary label. Combined with the ledger entries that supported it (the input probabilities), you have a full probabilistic-reasoning trace.

## Accepting

Acceptance works the same as it does for native or PyReason candidates:

```python
for cand in candidates:
    result = sdk.accept(cand, approved_by="demo", note="problog demo")
    print(f"Accepted: {result}")
```

The probability is preserved as confidence metadata on the resulting ledger assertion. The reasoning trace ProbLog produced becomes part of the audit story:

```python
persist_problog_annotations(sdk, candidates)
```

That second call writes the engine-specific reasoning trace alongside the regular evidence — proofs, probability calculations, branch resolutions. When a reviewer later asks *"why does the system think Alice's expertise probability is 0.72?"*, the trace is available, not just the final number.

## Verifying

The accepted facts are now in the ledger like any other:

```python
alice = sdk.get(Researcher, researcher_id="r-001")
print(alice.expertise)
# (whatever ProbLog inferred, with its probability as confidence)
```

If you read the assertion metadata, the probability is there:

```python
asrt = alice.assertions.expertise.active[0]
print(asrt.value, asrt.meta.get("confidence"))
```

Same shape as any other derived fact, with the difference that the confidence is grounded in real probabilistic reasoning rather than a heuristic.

## What ProbLog buys you

A few things that the native engine simply cannot give you, principally:

**Joint probabilities.** When two probabilistic facts both feed into a conclusion, ProbLog combines them properly. The native engine treats confidence labels as opaque; ProbLog treats them as probabilities and produces correct joint probability for the conclusion.

**Marginalisation.** *"What's the probability of `X`, integrating over all the ways it could be true?"* is a question ProbLog answers natively. The native engine has no notion of integrating over alternatives.

**Probabilistic disjunction.** Rules with multiple branches — *"the system became unhealthy because either A happened (with prob p) or B happened (with prob q)"* — combine into the right marginal under ProbLog. Under the native engine, the best you can do is take whichever fired and label it.

**Learning.** ProbLog has machinery for learning probabilities from data. We don't cover it in this tutorial, but the same `Derivation` shape feeds into ProbLog's learning entry points — useful when you have labelled outcomes and want to fit rule probabilities rather than hand-tune them.

## What ProbLog costs you

**Operational complexity.** ProbLog is a heavier dependency than the native engine. The `factpy-kernel[problog]` extra brings in the solver and its dependencies; in production you have to manage them.

**Speed.** Probabilistic inference is harder than logical inference. For large fact bases, ProbLog can be substantially slower than the native engine. For genuinely probabilistic problems this is unavoidable; just don't reach for ProbLog when the native engine is enough.

**Modelling care.** Garbage probabilities in produces garbage probabilities out, and the result still *looks* like a sober probabilistic answer. ProbLog won't catch the mistake of feeding in poorly-calibrated input probabilities. Probabilistic reasoning amplifies whatever quality you bring to the inputs, in both directions.

## Recap

In this tutorial you've:

- written facts with `confidence` metadata that ProbLog reads as observation probabilities,
- declared a `Derivation` with `mode="problog"` and a `ProbLogRuleExt` carrying engine config,
- evaluated under ProbLog with a timeout and inspected the resulting probabilistic candidates,
- accepted candidates back into the ledger with probabilities preserved as confidence,
- persisted ProbLog-specific reasoning traces as part of the audit story.

ProbLog is the right tool when the reasoning *is* probabilistic — not when you happen to have a confidence number you'd like to carry around. For the latter, the native engine with weighted bodies is simpler and fast enough.

## Where to next

- [Tutorial 05 — Onboarding journey](05-onboarding-journey.md) for an end-to-end scenario weaving native rules, derivations, and adapters.
- [Using adapters guide](../guides/using-adapters.md) for the comparative reference of all three adapter engines.
- [Audit and provenance](../concepts/audit-and-provenance.md) for how engine-specific traces (here, ProbLog proofs) fit into the broader audit story.