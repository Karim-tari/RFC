# LMDB Datastore

## Metadata

This proposal is (tick applicable):

* [ ] A new feature
* [x] an extension to [base layer architecture](./181107-base-layer-architecture.md)
* [ ] an existing feature or proposal

### Change log

### Status

| Date       | Status    |
|:-----------|:----------|
| 2018-11-01 | Submitted |


### Goals

Describe a generic interface for key-value object persistence for use by Tari domain objects.

### Assumptions
1. Storage requirements are served by a key-value storage solution
2. It is desirable that the specific storage implementation should be able to be changed without affecting the code that
   runs it.

### Abstract
A generic API is defined for key-value stores.

A specific implementation, based on LMDB is provided.


## Description

To prevent this documentation becoming stale, the interfaces are defined in code in [this PR](tbd)
