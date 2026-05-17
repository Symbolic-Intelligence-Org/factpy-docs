---
title: "factgraph"
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

# factgraph

`factgraph` is a Python library for append-only facts, rule-based inference, and auditable reasoning.

It is designed for applications that need to know not only what is currently true, but also where a fact came from, which assertion supports it, which rule derived it, and how it can be reviewed or explained.

## Why factgraph?

Most application state is mutable: a value changes, and the previous state disappears unless a separate audit system is added later.

`factgraph` starts from a different model. Facts are written as assertions into an append-only ledger. Current state is resolved from active assertions. Rules and inferences can reason over those facts, propose new facts, and preserve the evidence needed to inspect the result.

```text
schema -> assertions -> snapshots -> inference candidates -> accepted facts -> evidence
```

This keeps storage, reasoning, and audit connected instead of treating them as separate systems.