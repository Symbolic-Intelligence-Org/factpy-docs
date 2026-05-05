---
title: "FactPy"
type: docs
# bookFlatSection: false
# bookToc: false
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# FactPy

## The Problem

Most systems that hold facts about the world cannot give an account of how they came to hold them.

A row appears in a database. Where did it come from? An import job, three months ago, written by someone who left. A column was added; the old rows are silent about it. A pipeline ran and tagged customers as `high-risk`; you can no longer find which rule was used, on which version of the data, against which definition of *high-risk*. A model produced a score; the score is in the table; the inputs are gone.

When something goes wrong — a regulator asks why, a colleague spots a bad conclusion, a downstream system depends on a fact that turns out to be untrue — the reconstruction is forensic. You read logs that record what *happened* but not what was *concluded*. You re-run the pipeline and hope it produces the same answer. You discover that the rule was changed two weeks ago and nobody bumped a version. You write a post-mortem that mostly says *we couldn't tell*.

This is not a niche failure mode. It's the default state of every system that lets facts accumulate without recording how they got there. Schema migrations make it worse. ML inference makes it worse. Multi-source data merges make it worse.

The pieces of an answer exist. Append-only logs preserve history. Knowledge graphs preserve relationships. Rule engines preserve logic. Provenance vocabularies preserve attribution. But each of these solves one slice; nothing in common use binds them together so that *every fact a system holds carries its own justification*, mechanically, without ceremony, across schema changes, across reasoning engines, across team handoffs.

## What factpy does

factpy is a deterministic reasoning middleware that holds facts the way you'd hold them if accountability were the default rather than something you bolt on later.

The core is an append-only **ledger** of typed assertions. Every write is an entry: subject, predicate, value, time, source, trace id. Nothing is overwritten. Corrections are new entries; mistakes are retractions; the original is still there, still queryable, still part of the audit trail. The current state of any entity is a *projection* of the ledger at a chosen view — read-only, consistent, recomputed from the entries.

Schemas are typed and versioned. Each entity has named identity coordinates and named fields with declared cardinalities and type domains. The schema's content hash travels with the ledger file; reopening a stored ledger under an incompatible schema fails loudly rather than silently misreading old data. Adding a field is safe; renaming one isn't, and the system tells you so.

Rules are first-class. You write them in a small Python-embedded DSL, give them ids and versions, and they become objects you can run, compose, inspect, and ship. A *query* asks the ledger something. A *derivation* asks it something *and proposes new facts based on the answer*. Derivations don't write directly; they produce **candidates** with full evidence — the rule that fired, its version, the supporting ledger entries — which a separate `accept` step turns into ledger writes. *"every fact in the ledger is either directly written or accepted by an identifiable decision"* is the property that holds.

The whole shape of a run — writes, candidates, decisions, evidence — is exportable as a self-contained **audit package**. A directory of plain files. Diff-able, archive-able, shareable. Read it back with `kernel.audit`, walk from a high-level decision down to the leaf assertions that supported it, in code or in a UI. No running store needed. The bundle stands on its own.

When the reasoning shape itself outgrows the native evaluator — graph propagation through typed relationships, probabilistic inference over uncertain facts, large-scale Datalog — factpy hands off to engine adapters (PyReason, ProbLog, Souffle) without changing the schema, the DSL, or the audit story. The same `Derivation` object runs under a different engine; the candidates come back; the evidence is preserved.

## What working with it looks like

You declare entities like dataclasses. You write facts with `sdk.set` or `sdk.batch`. You read them with `sdk.get` or `sdk.find`. You write rules in a small DSL with `vars`, `Pred`, `Not`. You evaluate derivations to get candidates, review them, accept the ones you trust. When something goes wrong or someone needs to review the run, you call `sdk.export_package` and ship the directory.

That's the whole shape. The rest of the docs show what each move costs, what it buys, and where the edges are.

## Where to start

If you'd rather read concepts before writing code, start with the [overview](concepts/overview.md) and follow the concept track.

If you'd rather build something and see it work, jump to the [quickstart](quickstart.md) or [tutorial 01](tutorials/01-sdk-basics.md).

If you have a specific task in mind — defining a schema, writing rules, producing an audit package — the [guides](guides/_index.md) are organized by task.

If you want the API surface, every entry point is in the [reference](reference/_index.md).