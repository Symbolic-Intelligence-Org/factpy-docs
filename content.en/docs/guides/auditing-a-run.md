---
title: "Auditing a Run"
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

# Auditing a run

Every run of factpy — writes, queries, derivation evaluations, accepts — leaves records in the store as it goes. Producing an audit package is the act of *bundling those records into a self-contained directory* you can ship, archive, or read elsewhere. This guide walks through producing one, what's on disk afterwards, and how to read it back. For the conceptual picture, see [audit and provenance](../concepts/audit-and-provenance.md).

## Producing a package

After a run — a batch of writes, a derivation evaluated and partly accepted, a query executed — call `sdk.export_package` with an output directory and `ExportOptions(package_kind="audit")`:

```python
from pathlib import Path
from kernel.adapters.souffle.package import ExportOptions

out_dir = Path("./audits/run-2024-04-01")
sdk.export_package(out_dir, ExportOptions(package_kind="audit"))
```

`package_kind="audit"` selects the audit shape; the alternative, `"inference"`, produces a leaner package meant for re-running rather than for review. For audit purposes, always use `"audit"` — it includes the run ledger, candidate ledger, accept-write ledger, decision log, and any evidence graphs that were produced during the run.

The export does not modify the store. You can produce a package mid-session, after every commit, or once at the end of a long-running job. Multiple packages from the same store are independent; each is a snapshot of the run records as they stood at export time.

## Anatomy of a package

The directory `sdk.export_package` writes is structured the same way every time:

```
audits/run-2024-04-01/
├── manifest.json
├── schema/
│   └── schema_ir.json
├── policy/
├── facts/
├── rules/
├── outputs/
│   └── run_manifest.json
└── audit/
    ├── run_ledger.jsonl
    ├── candidate_ledger.jsonl
    ├── accept_write_ledger.jsonl
    ├── accept_failed.jsonl
    ├── decision_log.jsonl
    ├── mapping_resolution.json
    ├── support_artifacts.jsonl       (optional)
    ├── rule_trace_artifacts.jsonl    (optional)
    └── evidence_graphs/              (optional, per-candidate)
```

`manifest.json` is the entry point. It declares the package kind, lists the relative paths of every audit file, and carries the schema and rule versions in use. An audit reader should always start there rather than walking the directory blind — the manifest is the source of truth for what the package contains.

The `audit/` subdirectory is where the run records live. Each `*.jsonl` file is a stream of JSON-encoded entries — one per line — covering one slice of the run: every write, every candidate, every accept, every decision, every failure. Optional artifacts (rule traces, evidence graphs) appear only when the run produced them.

The other top-level directories (`schema/`, `rules/`, `facts/`, `outputs/`) hold the materialized run inputs and outputs — useful when you want to re-run the package under a different engine, but not strictly required for reading the audit story.

## Reading a package back

`kernel.audit` reads packages without any of the rest of factpy. It's a small module by design: the dependency surface for someone reviewing an audit is much smaller than for someone *producing* one.

```python
from kernel.audit import load_audit_package

pkg = load_audit_package("./audits/run-2024-04-01")

print(pkg.manifest["package_kind"])         # "audit"
print(len(pkg.run_ledger))                  # writes during the run
print(len(pkg.candidate_ledger))            # candidates produced
print(len(pkg.accept_write_ledger))         # candidates accepted
print(len(pkg.decision_log))                # decisions recorded
print(len(pkg.accept_failed))               # accepts that failed
```

`load_audit_package` returns an `AuditPackageData` — a dataclass whose attributes are the parsed contents of every file in the package. The `*_ledger` attributes are lists of plain dicts (jsonl-decoded). The `evidence_graphs` attribute is a dict of `EvidenceGraph` objects keyed by candidate. The `manifest` and `run_manifest` attributes are the parsed JSON.

From there, you build whatever view you need.

## Common views

The kernel ships builder functions for the views audit consumers most often want. They're useful both as ready-made navigation and as documentation of how to slice the data yourself.

For a per-decision walkthrough — *"what was accepted, by whom, supported by what"* — use `build_decision_detail_dto`:

```python
from kernel.audit import build_decision_detail_dto

for entry in pkg.decision_log:
    dto = build_decision_detail_dto(pkg, decision_id=entry["decision_id"])
    # dto carries the candidate, the accepted assertion, and the evidence
```

For a per-rule trace — *"which rule fired, against which ledger state, returning what"* — use `build_rule_trace_detail_dto`. For a candidate's full evidence tree (or graph, for adapter-produced candidates), use `build_candidate_evidence_tree_dto`. For a high-level run summary, `build_run_detail_dto`.

The full list lives in `kernel.audit.__all__`; each builder takes the `AuditPackageData` and a key identifying the slice you want.

## What a reviewer asks, and where to look

A few common questions and the data that answers them.

*"Which facts were written by this run?"* — `pkg.run_ledger`. Each entry is one assertion, with its predicate, subject, value, and metadata.

*"Which of those came from a derivation rather than direct writes?"* — `pkg.accept_write_ledger`. These are the writes that originated as accepted candidates; each carries a reference back to the candidate and rule.

*"Why was a particular fact accepted?"* — find the `decision_id` in `pkg.decision_log`, then the candidate it accepted, then the candidate's evidence in `pkg.provenance_trees` or `pkg.evidence_graphs`.

*"Which candidates were proposed but not accepted?"* — `pkg.candidate_ledger` minus what's referenced from `pkg.accept_write_ledger`. The candidates that ended up unaccepted are visible — the package is honest about both directions.

*"What schema and rule versions were in use?"* — `pkg.manifest`, plus the materialized rule artifacts under `rules/` for the bodies themselves.

The decision log is the connective tissue. Every accept produces a log entry; every log entry references the candidate and the resulting writes. Walk it forwards from candidates to writes, or backwards from writes to candidates, depending on what you want to show.

## Sharing a package

The package is a directory of plain files. You can:

- archive it with `tar` or `zip` and ship the archive,
- commit it to a separate audit repository,
- write it to object storage as a versioned bundle,
- diff two packages with normal text tools, since the jsonl is line-stable.

A reviewer needs only `kernel.audit` — and only the audit subset of factpy — to read it back. They do not need a running store, a ledger file, or any of the schema/rule code. This is the property that makes audit packages a deliverable rather than a debugging aid: the bundle stands on its own.

## When to export

Two reasonable rhythms.

**Per logical run.** A batch import, a nightly derivation pass, a reviewer's accept session — each of these is one logical run, and exporting a package at its end gives you one bundle to point at when someone asks *what happened on April 1st*. This is the typical case.

**Per release, plus on demand.** A long-running service exports a baseline package at each release boundary and an incremental package whenever an investigation requires one. The baselines are archived; the incrementals are produced and shared as needed.

For ad-hoc development or tests, you usually don't need to export at all — the run records exist in the store regardless. Export is for when the records need to leave the system that produced them.

## Where to next

- [Audit and provenance](../concepts/audit-and-provenance.md) for the conceptual picture of what's recorded and why.
- The (when written) [reference for `kernel.audit`](../reference/audit.md) covers every builder DTO, the evidence-graph node/edge taxonomy, and the manifest schema in full.
- [Running derivations](running-derivations.md) for how the candidate-and-accept records that fill an audit package come into being in the first place.