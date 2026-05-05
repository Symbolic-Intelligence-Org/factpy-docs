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

Every operation a factpy system performs — writes, queries, derivation evaluations, acceptances — leaves records in the store as it goes. Producing an audit package is the act of bundling those records into a self-contained directory that can be shipped, archived, or reviewed elsewhere, independently of the system that produced it. The present guide develops the mechanics of producing such a package, the structure that the package has on disk, the way `kernel.audit` reads it back, the views that an audit reader most often constructs from it, and the rhythms in which export is sensibly performed. For the conceptual material on which the package format rests, see [audit and provenance](/docs/concepts/audit-and-provenance).

## Producing a package

A package is produced by calling `sdk.export_package` with an output directory and an `ExportOptions` object that selects the package shape.

```python
from pathlib import Path
from kernel.adapters.souffle.package import ExportOptions

out_dir = Path("./audits/run-2024-04-01")
sdk.export_package(out_dir, ExportOptions(package_kind="audit"))
```

The `package_kind="audit"` setting selects the audit shape, which includes the run ledger, the candidate ledger, the accept-write ledger, the decision log, and any evidence graphs that were produced during the run. The alternative, `"inference"`, produces a leaner package suitable for re-execution under another engine but without the audit-side records that make a package interpretable in review. For audit purposes the `"audit"` shape is always the appropriate choice.

The export does not modify the store. A package can be produced mid-session, after every commit, or once at the end of a long-running job, and multiple packages from the same store are independent of one another, each being a snapshot of the run records as they stood at the moment of export.

## Anatomy of a package

The directory that `sdk.export_package` writes is structured the same way every time, with a top-level manifest and several subdirectories that separate the run records from the materialised inputs.

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

`manifest.json` is the entry point. It declares the package's kind, lists the relative paths of every audit file, and carries the schema and rule versions in use during the run. An audit reader is intended to start there rather than walking the directory blind, since the manifest is the authoritative index of what the package contains and the file against which any subsequent navigation is best resolved.

The `audit/` subdirectory is where the run records themselves live. Each `*.jsonl` file is a stream of JSON-encoded entries, one per line, covering one slice of the run: every write, every candidate, every acceptance, every decision, and every accept that failed. The optional artifacts — rule traces, per-candidate evidence graphs — appear only when the run produced them and are referenced from the manifest where present.

The remaining top-level directories — `schema/`, `rules/`, `facts/`, `outputs/` — hold the materialised inputs and outputs of the run. They are useful when the package is to be re-executed under a different engine, but they are not strictly required for reading the audit story; an audit reader interested only in what happened during the run can confine its attention to `manifest.json` and the `audit/` subdirectory.

## Reading a package back

`kernel.audit` reads packages without depending on the rest of factpy. The module is small by design, since the dependency surface required to review an audit is far smaller than the surface required to produce one, and the audit story benefits from being readable on machines that have nothing else of factpy installed.

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

`load_audit_package` returns an `AuditPackageData` instance, a dataclass whose attributes are the parsed contents of every file in the package. The `*_ledger` attributes are lists of plain dicts decoded from jsonl, the `evidence_graphs` attribute is a dictionary of `EvidenceGraph` objects keyed by candidate, and the `manifest` and `run_manifest` attributes are the parsed JSON of those two files. From these, an application or a reviewer constructs whatever view the situation calls for.

## Common views

The kernel ships builder functions for the views that audit consumers most often want. They serve two purposes simultaneously: as ready-made navigation surfaces for applications that do not want to write their own, and as worked examples of how each slice of the package is to be assembled by hand when a custom view is wanted.

For a per-decision walkthrough — *what was accepted, by whom, supported by what* — `build_decision_detail_dto` is the appropriate builder.

```python
from kernel.audit import build_decision_detail_dto

for entry in pkg.decision_log:
    dto = build_decision_detail_dto(pkg, decision_id=entry["decision_id"])
    # dto carries the candidate, the accepted assertion, and the evidence
```

For a per-rule trace — *which rule fired, against which ledger state, returning what* — `build_rule_trace_detail_dto` is the appropriate builder. For a candidate's full evidence tree, or the corresponding graph for adapter-produced candidates, `build_candidate_evidence_tree_dto` is appropriate. For a high-level summary of the run as a whole, `build_run_detail_dto` is appropriate. The full list of builders is enumerated in `kernel.audit.__all__`, and each builder takes an `AuditPackageData` instance together with a key identifying the slice it is to extract.

## What a reviewer asks, and where to look

A handful of questions recur whenever an audit package is opened, and the data that answers each is in a known place.

*Which facts were written by this run?* The answer is `pkg.run_ledger`. Each entry is one assertion, carrying its predicate, subject, value, and metadata.

*Which of those came from a derivation rather than from direct writes?* The answer is `pkg.accept_write_ledger`. These are the entries whose origin was an accepted candidate, and each carries a reference back to the candidate and the rule that produced it.

*Why was a particular fact accepted?* The path is from `pkg.decision_log` to the candidate it accepted, and from the candidate to its evidence in `pkg.provenance_trees` or `pkg.evidence_graphs`.

*Which candidates were proposed but not accepted?* The answer is `pkg.candidate_ledger` minus the candidates referenced from `pkg.accept_write_ledger`. The package is honest about both directions: candidates that were considered but ultimately not accepted remain visible, and the audit reader can recover not only what the system did but also what it deliberately declined to do.

*What schema and rule versions were in use?* The answer is `pkg.manifest`, supplemented by the materialised rule artifacts under `rules/` for the bodies themselves.

The decision log is the connective tissue between candidates and writes. Every acceptance produces a log entry, and every log entry references both the candidate it accepted and the writes that resulted; the log can therefore be walked forwards from candidates to the writes they produced, or backwards from writes to the candidates that licensed them, depending on the direction in which a particular question is most naturally answered.

## Sharing a package

A package is a directory of plain files, and the means of sharing it are correspondingly ordinary. It can be archived with `tar` or `zip` and the archive shipped through any channel; it can be committed to a separate audit repository under version control; it can be written to object storage as a versioned bundle; it can be diffed between two runs with ordinary text tools, since the jsonl files are line-stable.

A reviewer requires only `kernel.audit` and the audit subset of factpy in order to read a package back. There is no requirement of a running store, no ledger file to be supplied alongside the package, and no need for the schema or rule code that produced the run. This self-containment is what makes audit packages a deliverable rather than a debugging aid: the bundle stands on its own, interpretable independently of the system that produced it.

## When to export

Two rhythms of export are appropriate to most use cases.

The first is *export per logical run*. A batch import, a nightly derivation pass, a reviewer's accept session — each of these is one logical unit of work, and exporting a package at the end of each gives a single bundle to point at when someone later asks what happened during it. This is the typical case for systems whose work falls into discrete runs.

The second is *baseline plus on-demand export*. A long-running service exports a baseline package at each release boundary, and produces an additional package whenever an investigation requires one. The baselines are archived as the system's audit history; the on-demand packages are generated and shared as questions arise.

For ad-hoc development and for tests, exporting is generally unnecessary. The run records exist in the store regardless of whether they are exported, and the export is only required when the records need to leave the system that produced them.

## Where to next

[Audit and provenance](/docs/concepts/audit-and-provenance) develops the conceptual picture of what is recorded during a run and why. The reference for `kernel.audit` covers every builder DTO, the evidence-graph node and edge taxonomy, and the manifest schema in full. [Running derivations](/docs/guides/running-derivations) develops the candidate-and-acceptance lifecycle whose records fill an audit package in the first place.