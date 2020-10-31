---
layout: post
title: SQLite commit protocol
---

# Introduction

SQLite commit protocol is separated in two phases.

- First phase: the journal/wal file will be updated, and all dirty pages will be synced
- Second phase: if there is any issue when committing the 1st phase, this phase will rollback the database. Else, delete/truncate/clean the journal file (if the database is running at journal mode)

The detail implementation will be described below:

## First phase

### Function specification

`btree.c` - `int sqlite3BtreeCommitPhaseOne(Btree *p, const char *zSuperJrnl)`

Inputs:
- `Btree *p`: The b+tree data structure want to flush to disk
- `const char *zSuperJrnl`: The name of the journal file

`pager.c` - `int sqlite3PagerCommitPhaseOne(Pager *pPager, const char *zSuper, int noSync`

Inputs:
- `Pager *pPager`: The pager object of the current `BtShared` - `Btree *p->pBt`
- `const char *zSuper`: The name of the journal file
- `int noSync`: false if all dirty pages should be synced immediately in this function. Else, engineer has to sync it on his own

### Detail implementation

- If the current transaction is not a write one, then simply return `SQLITE_OK`
- Lock the whole `Btree *p`
  - `sqlite3BtreeEnter(p)`
- Execute `sqlite3PagerCommitPhaseOne` - instruct the pager to sync all dirty pages automatically
  - `sqlite3PagerCommitPhaseOne(pBt->pPager, zSuperJrnl, 0)`
- Unlock the `Btree *p` and return

### Pager commit phase one

- The function will exit early if there is no made change in current transaction
- If the current pager uses WAL
  - Log the dirty pages to WAL
    - `rc = pagerWalFrames(pPager, pList, pPager->dbSize, 1);`
  - Clean all the dirty pages
    - `sqlite3PcacheCleanAll(pPager->pPCache);`
- Else
  - As `SQLITE_ENABLE_BATCH_ATOMIC_WRITE` is disabled by default (it is only available in F2FS filesystem), we don't have to care about the logic in the `#ifdef SQLITE_ENABLE_BATCH_ATOMIC_WRITE` block
  - `SQLITE_ENABLE_ATOMIC_WRITE` should be enabled
  - Increase file change counter of current database - inside the `#ifdef SQLITE_ENABLE_ATOMIC_WRITE` block
  - Write Metadata to the current journal file
    - `rc = writeSuperJournal(pPager, zSuper);`
  - Sync the current journal file and write all dirty pages to the database
    - `rc = syncJournal(pPager, 0);`
  - Get the current dirty pages as list, write them and clean them all
    - `pList = sqlite3PcacheDirtyList(pPager->pPCache);`
    - `rc = pager_write_pagelist(pPager, pList);`
    - `sqlite3PcacheCleanAll(pPager->pPCache);`
  - Fix the case when the file on disk is smaller than the database image
    - `if( pPager->dbSize>pPager->dbFileSize ) { ... }`
  - Sync the database file
    - `rc = sqlite3PagerSync(pPager, zSuper);`
  - Mark the state of pager to `FINISHED`
    - `pPager->eState = PAGER_WRITER_FINISHED;`