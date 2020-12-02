---
layout: post
title: SQLite commit protocol
tags: [sqlite]
---

# Introduction

SQLite commit protocol is separated into two phases.

- First phase: the journal/WAL file will be updated, and all dirty pages will be synced
- Second phase: if there is an issue when committing the 1st phase, this phase will rollback the database. Else, delete/truncate/clean the journal file (if the database is running at journal mode)

The detail implementation will be described below:

## First phase

### Function specification

#### BTree commit phase one API

`btree.c` - `int sqlite3BtreeCommitPhaseOne(Btree *p, const char *zSuperJrnl)`

Inputs:
- `Btree *p`: The b+tree data structure want to flush to disk
- `const char *zSuperJrnl`: The name of the journal file

#### Pager commit phase one API

`pager.c` - `int sqlite3PagerCommitPhaseOne(Pager *pPager, const char *zSuper, int noSync`

Inputs:
- `Pager *pPager`: The pager object of the current `BtShared` - `Btree *p->pBt`
- `const char *zSuper`: The name of the journal file
- `int noSync`: false if all dirty pages should be synced immediately in this function. Else, the engineer has to sync it on his own

#### List of dirty pages in Pager

`pcache.c` - `PgHdr *sqlite3PcacheDirtyList(PCache *pCache)`

**Description**: Return a list of all dirty pages in the cache, sorted by page number.

Inputs:
- `PCache *pCache`: Current `PCache` object in the `Pager`

Outputs:
- A linked list of dirty pages - `PgHdr *`, sorted by pageno

### Detail implementation

- If the current transaction is not a write one, then simply return `SQLITE_OK`
- Lock the whole `Btree *p`
  - `sqlite3BtreeEnter(p);`
- Execute `sqlite3PagerCommitPhaseOne` - instruct the pager to sync all dirty pages automatically
  - `rc = sqlite3PagerCommitPhaseOne(pBt->pPager, zSuperJrnl, 0)`
- Unlock the `Btree *p` and return

### Pager commit phase one

- The function will exit early if there is no made change in the current transaction
- If the current pager uses WAL
  - Log the dirty pages to WAL
    - `rc = pagerWalFrames(pPager, pList, pPager->dbSize, 1);`
  - Clean all the dirty pages
    - `sqlite3PcacheCleanAll(pPager->pPCache);`
- Else
  - As `SQLITE_ENABLE_BATCH_ATOMIC_WRITE` is disabled by default (it is only available in F2FS filesystem), we don't have to care about the logic in the `#ifdef SQLITE_ENABLE_BATCH_ATOMIC_WRITE` block
  - `SQLITE_ENABLE_ATOMIC_WRITE` should be enabled
  - Increase file change counter of the current database - inside the `#ifdef SQLITE_ENABLE_ATOMIC_WRITE` block
  - Write Metadata to the current journal file
    - `rc = writeSuperJournal(pPager, zSuper);`
  - Sync the current journal file and write all dirty pages to the database
    - `rc = syncJournal(pPager, 0);`
  - Get the current dirty pages as a linked list, write them and clean them all
    - `pList = sqlite3PcacheDirtyList(pPager->pPCache);`
    - `rc = pager_write_pagelist(pPager, pList);`
    - `sqlite3PcacheCleanAll(pPager->pPCache);`
  - Fix the case when the file on disk is smaller than the database image
    - `if( pPager->dbSize>pPager->dbFileSize ) { ... }`
  - Sync the database file
    - `rc = sqlite3PagerSync(pPager, zSuper);`
  - Mark the state of the pager to `FINISHED`
    - `pPager->eState = PAGER_WRITER_FINISHED;`

## Second phase

### Function specification

#### BTree commit phase two API

`btree.c` - `int sqlite3BtreeCommitPhaseTwo(Btree *p, int bCleanup)`

Inputs:
- `Btree *p`: The b+tree data structure want to flush to disk
- `int bCleanup`: This argument is not important (for now)

#### Pager commit phase two API

`pager.c` - `int sqlite3PagerCommitPhaseTwo(Pager *pPager)`

Inputs:
- `Pager *pPager`: The pager object of the current `BtShared` - `Btree *p->pBt`

### Detail implementation

- The function will exit early if there is no made change in current transaction
- Lock the whole `Btree *p`
  - `sqlite3BtreeEnter(p)`
- Execute `sqlite3PagerCommitPhaseTwo` - instruct the pager to sync all dirty pages automatically
  - `rc = sqlite3PagerCommitPhaseTwo(pBt->pPager);`
- Compensate for pPager->iDataVersion++ and update the current txn to read txn
  - `p->iDataVersion--;`
  - `pBt->inTransaction = TRANS_READ;`
- Clear the list of pages recently moved to free-list in this transaction
  - `btreeClearHasContent(pBt);`
- Unlock the `Btree *p` and return

### Pager commit phase two

The detail implementation of this API is not important, as this is simply a journal cleaning phase