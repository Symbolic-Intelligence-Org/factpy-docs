---
title: "Persistence"
weight: 7
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Persistence

The default ledger lives in memory: it exists for the lifetime of the process that opened it and is discarded when the process exits. This is the appropriate behaviour for tests and for short-lived computation, and is what `SDKStore.from_schema_classes([...])` produces when no further arguments are supplied. For everything else — long-running services, jobs whose state must survive a restart, schemas that evolve over time — the ledger is persisted to a file. The present guide develops how persistence is opened, what the schema digest is and what it protects against, which kinds of schema change are safe across restarts and which require migration, how processes share a persistent ledger without losing the audit story, and how sidecar artifacts and backups relate to the ledger file itself.

## Opening a persistent store

Persistence is opt-in through a `ledger_path` argument supplied at the moment the store is opened.

```python
from kernel.sdk import SDKStore

sdk = SDKStore.from_schema_classes(
    [Person, Country, Order],
    ledger_path="./data/factpy.db",
)
```

The first time a store is opened with a given path, the file is created and the schema's digest is recorded into its metadata. Subsequent writes append entries to the file in the order they occur. When the same path is opened again, the digest of the schema being supplied is recomputed and compared against the digest stored in the file. A match permits the open to proceed and the ledger to be used for reads and writes; a mismatch fails the open with an explicit error rather than allowing the kernel to operate against incompatible data.

The ledger file is self-contained. The schema digest, the ledger entries, and any sidecar-artifact references all live within the path supplied at open time, and moving the file moves the store as a whole. There is no auxiliary configuration that must travel alongside the file for it to remain readable.

## What the schema digest protects

The digest is a content hash computed over the schema as the kernel sees it: every entity class, every identity field with its primary-key flag and type domain, every field with its cardinality and type domain, together with the `Meta.version` of each entity. Two schemas produce the same digest if and only if their structurally meaningful parts are identical. Renaming a python module, reordering the classes in the list passed to `from_schema_classes`, or adding a docstring to a class does not change the digest, since none of these alters what facts the schema admits or how those facts are addressed. Renaming a field, changing a cardinality, or swapping a type domain does change the digest, since each of these alters the legibility of any assertion the previous schema had written.

The check exists not in order to prevent edits but in order to ensure that when a ledger is reopened — possibly months after it was first written, possibly by a process that does not know its history — the schema being supplied at open time can read the ledger's facts coherently. Silently misreading older data under a changed schema is the failure mode the digest is constructed to rule out, and the failure-loud-on-mismatch behaviour is what gives the digest its value as a protection.

## Safe and unsafe changes

Some schema changes preserve legibility and require no digest bump and no migration. Adding a new entity class extends the vocabulary without affecting any existing assertion. Adding a new field to an existing entity introduces a predicate for which existing entities have no assertions, which is a normal and well-defined state under the projection rules. Adding optional metadata where none existed previously — descriptions, tags, an explicit version where the default was implicit — does not affect the digest. Adding a new `Identity` field with a `default` or `default_factory` allows existing identity tuples to be extended without rewriting them, since the kernel can populate the new coordinate at write time when none is supplied.

Other changes do not preserve legibility and constitute a new schema version. Renaming a field or an entity makes assertions about the old name unreadable under the new vocabulary. Changing a field's cardinality between `single` and `multi` alters how existing assertions reduce in projection. Changing a field's type domain — from `string` to `int`, or from a literal type to `entity_ref` — alters how existing values are interpreted. Changing the identity composition of an existing entity invalidates the addresses by which existing assertions are located. Removing a field that has been written to leaves orphaned assertions in the ledger that the new schema has no place for. Each of these changes alters the digest, the kernel refuses to open the old ledger under the new schema, and migration becomes an explicit step rather than an implicit reinterpretation.

The dividing line is legibility. An additive change extends the vocabulary without unsettling what the existing entries mean, and the kernel can proceed against an old ledger under the new schema. A change that alters the meaning of existing entries — by renaming the predicate they refer to, by altering the projection rule that reduces them, or by changing the addresses at which they are located — does unsettle their meaning, and the kernel will not pretend otherwise.

## Migrating a schema

When a change crosses the line from legible to illegible, migration is the explicit operation by which a new ledger is constructed under the new schema. The general shape involves four steps. The relevant entity's `Meta.version` is bumped, recording the change in the schema itself. The old ledger is opened under the old schema in a separate process or in-memory store, where its assertions remain accessible. Those assertions are replayed into a new ledger under the new schema, with whatever transformations the schema change requires applied as part of the replay. The service is then switched to the new ledger.

The kernel does not provide an in-place migration tool, by design. A migration is a small program that reads from one store and writes to another, and because the ledger is append-only, both ledgers can be kept in parallel for as long as the audit story needs to span the cut. For development work in which the schema is still molten and there is no data that must be preserved, deleting the file and reopening is the appropriate course; the digest check exists to protect data that exists, and where there is no such data it imposes no further constraint.

## Crossing process boundaries

A persistent ledger lets two processes that share the file see the same state, and two patterns of inter-process use are worth distinguishing.

The first is *sequential handoff*: process A writes, process A exits, process B opens the file and reads. This is the common case — a batch import that runs to completion, followed by a service that serves reads from the resulting ledger — and the ledger file is the entire interface between the two processes. Nothing further has to be coordinated, since there is no temporal overlap during which both processes hold the file open.

The second is *concurrent access*: two or more processes hold the same ledger file open simultaneously. The kernel's locking is sufficient for read-heavy workloads in which a single writer is occasional and the rest of the access is reads. For workloads with multiple concurrent writers, the appropriate shape is typically a single writer process accompanied by reader processes — or a shared service that mediates writes on behalf of multiple clients — rather than several independent processes appending in parallel. The ledger format itself is well-behaved under concurrent reads; the contention point is the metadata coordination that surrounds writes, and this is considerably easier to reason about with a single writer.

In both patterns the audit story is preserved: every assertion is timestamped, sourced, and traceable, regardless of which process produced it. The fact that some assertions came from a different process appears in the audit record as a piece of metadata like any other.

## Sidecar artifacts

Facts whose values are large — files, images, model weights — are best kept out of the ledger file itself, and the kernel supports this through an `artifact_store_root` argument that designates a directory where the binary content is stored separately.

```python
sdk = SDKStore.from_schema_classes(
    [Person, Document],
    ledger_path="./data/factpy.db",
    artifact_store_root="./data/artifacts",
)
```

Large values are written into the directory under `artifact_store_root` and the ledger holds a content-addressed reference rather than the value itself. The audit story is preserved in the same form as for any other fact — the reference is part of the assertion, the file remains reachable from the package — without bloating the ledger file with binary content.

When `artifact_store_root` is not supplied, large values flow through the ledger directly, which is acceptable for development work and small artifacts but generally inappropriate for production deployments handling sizable blobs. The sidecar configuration is then worth setting up.

## Backups

Backing up a factpy store consists of backing up two things: the ledger file at `ledger_path` and, when one has been configured, the directory at `artifact_store_root`. Both are append-only at the kernel level — the ledger never rewrites earlier entries, the sidecar never rewrites earlier blobs — and a copy taken while the store is quiescent is therefore internally consistent without any further coordination.

For a backup of a running store, the simplest correct procedure is to quiesce writes briefly, copy the file together with the sidecar directory if one is in use, and resume. The resulting copy is a complete snapshot from which the store can be replayed or reopened elsewhere.

Audit packages produced through `sdk.export_package` are an additional archival format, not a substitute for ledger backups. They capture the records of one or more runs — what was written, what was proposed, what was accepted — and are the appropriate artifact for review and offline inspection. They are not, however, sufficient as a backup of operational state: a ledger file is what one opens to continue operating against the data, and the audit package is what one opens to inspect what was done. Both are worth retaining, in different cadences and for different purposes.

## Where to next

[Defining a schema](/docs/guides/defining-a-schema) develops the rules around what changes require a new schema version. [Auditing a run](/docs/guides/auditing-a-run) covers how audit packages relate to ledger files and when each is the appropriate artifact. For the underlying append-only model and the mechanics by which snapshots are computed from a persisted ledger, see [the ledger](/docs/concepts/the-ledger).