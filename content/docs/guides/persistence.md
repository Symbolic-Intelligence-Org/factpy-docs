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

The default ledger lives in memory: it exists for the lifetime of the process and disappears when the process exits. That's the right behaviour for tests and for short-lived computation, and it's what you get when you call `SDKStore.from_schema_classes([...])` with no extra arguments.

For everything else — long-running services, jobs whose state has to survive a restart, schemas you want to evolve over time — you persist the ledger to a file. This guide covers how to do that, what the schema digest is, what kinds of changes are safe across restarts, and how to cross process boundaries without losing the audit story.

## Opening a persistent store

Pass `ledger_path` to `from_schema_classes`:

```python
from kernel.sdk import SDKStore

sdk = SDKStore.from_schema_classes(
    [Person, Country, Order],
    ledger_path="./data/factpy.db",
)
```

The first time you open a store with a given path, the file is created and the schema's digest is written into its metadata. Subsequent writes append to the file in the order they happen.

The next time you open the same path, the SDK re-computes the digest of the schema you're opening with and compares it to the one stored in the file. If they match, the open succeeds and the ledger is ready to use. If they don't, the open fails with a clear error rather than letting you proceed against incompatible data.

The file is self-contained: schema digest, ledger entries, and any sidecar artifact references all live inside the path you gave. Move the file, you've moved the store.

## What the schema digest protects

The digest is a content hash over the schema as the SDK sees it: every entity, every identity field with its primary-key flag and type domain, every field with its cardinality and type domain, plus the `Meta.version` of each entity.

Two schemas produce the same digest iff their structurally meaningful parts are identical. Renaming a python module, reordering classes in the list passed to `from_schema_classes`, adding a docstring — none of these change the digest. Renaming a field, changing a cardinality, swapping a type domain — all of these do.

The point of the check is not to prevent edits. It's to make sure that when you reopen a ledger written months ago, the schema you're opening it with can still *legibly* read those facts. Silently misreading old data is the failure mode the digest exists to rule out.

## Safe and unsafe changes

What's safe across restarts — meaning, no digest bump, no migration:

- adding a new entity class,
- adding a new field to an existing entity,
- adding optional metadata (`Meta.description`, `Meta.tags`) where there was none,
- adding a new `Identity` field with a `default` or `default_factory`.

What requires a new schema version — meaning, a digest change, the kernel will refuse to open the old ledger with the new schema, and you have to migrate explicitly:

- renaming a field or an entity,
- changing a field's cardinality (`single` ↔ `multi`),
- changing a field's type domain (`string` → `int`, `string` → `entity_ref`, ...),
- changing the identity composition of an existing entity,
- removing a field that has been written to.

The dividing line is *legibility*. Adding a field doesn't break old facts; old entities just have nothing for the new field, which is a normal and well-defined state. Renaming a field makes the old assertions unreadable: the system has facts about a predicate the schema no longer knows. The kernel won't pretend that's fine.

## Migrating a schema

When you need to change something that crosses the safe line, the migration shape is roughly:

1. Bump the relevant entity's `Meta.version`.
2. Open the *old* ledger under the *old* schema in a separate process or in-memory store.
3. Replay its assertions into a *new* ledger under the *new* schema, transforming as you go.
4. Switch your service to the new ledger.

There is no in-place migration tool in the kernel by design. A migration is a small program that reads from one store and writes to another — and because the ledger is append-only, you can keep both around for as long as you need the audit story to span the cut.

For development work where the schema is still molten and there is no production data, migrations are overkill. Delete the file, reopen, move on. The digest check exists to protect *data that exists*; nothing else.

## Crossing process boundaries

A persistent ledger lets two processes that share the file see the same state. There are two patterns worth distinguishing.

**Sequential handoff.** Process A writes; process A exits; process B opens the file and reads. This is the common case — a batch import that runs to completion, followed by a service that serves reads. The ledger file is the entire interface; nothing else has to be coordinated.

**Concurrent access.** Two processes hold the same ledger file open at the same time. The kernel's locking is sufficient for read-heavy workloads with occasional writes from a single writer. For multiple concurrent writers, the right shape is usually a single writer process plus reader processes — or a shared service that mediates writes — rather than two independent processes both appending. The ledger format itself is happy with concurrent reads; the contention point is metadata coordination, which is easier to reason about with a single writer.

In both patterns, the audit story doesn't change: every assertion is still timestamped, sourced, and traceable. The fact that some assertions came from a different process is just another piece of metadata.

## Sidecar artifacts

For facts that reference large blobs — files, images, model weights — pass `artifact_store_root` to keep them out of the ledger:

```python
sdk = SDKStore.from_schema_classes(
    [Person, Document],
    ledger_path="./data/factpy.db",
    artifact_store_root="./data/artifacts",
)
```

Large values are stored under `artifact_store_root` and the ledger holds a content-addressed reference. The audit story is preserved — the reference is part of the assertion, the file is reachable from the package — without bloating the ledger file itself.

If you don't pass `artifact_store_root`, large values flow through the ledger directly. That's fine for development and small artifacts; for production with sizable blobs, the sidecar is worth setting up.

## Backups

Backing up a factpy store means backing up two things: the ledger file at `ledger_path`, and (if used) the directory at `artifact_store_root`. Both are append-only at the kernel level — the ledger never rewrites earlier entries, the sidecar never rewrites earlier blobs — so a copy taken while the store is quiescent is consistent.

For a backup of a running store, the simplest correct approach is to quiesce writes briefly, copy the file (and the sidecar directory if any), and resume. The copy is then a complete snapshot you can replay or open from elsewhere.

Audit packages produced via `sdk.export_package` are an *additional* archival format, not a backup. They capture run records — what was written, what was proposed, what was accepted — but they are not a substitute for the ledger file when you want to keep operating against the data. Keep both: ledger files for state, audit packages for review.

## Where to next

- [Defining a schema](defining-a-schema.md) for the rules around what changes require a new schema version.
- [Auditing a run](auditing-a-run.md) for how packages relate to ledger files and when each is the right artifact.
- For the underlying append-only model and how snapshots are computed from a persisted ledger, see [the ledger](../concepts/the-ledger.md).