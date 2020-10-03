---
layout: post
title: SQLite pager module
---

## Problems when working with SQLite pager module

Some of the most important structs of `pager` module are defined in source files (files with .c extension), such as `struct Pager`, `struct PCache`, which makes the experiment complicated than expectation. One of the struggles that bothered me the most is it is impossible to access the `pager->pPCache` object.

## Important data structures and functions
- `pager.c` - `struct Pager`
- `pcache.c` - `struct PCache`
- `pcache.h` - `struct PgHdr` (aka. `pager.h` - `struct DbPage`)
- `pcache1.c` - all functions
  - they are the default implementation of the `sqlite3_pcache` interface
  - the interface is known as `sqlite3_pcache_methods2`
  - the functions are mostly used by calling `sqlite3GlobalConfig.pcache2.*`

## Pager module behavior

It is important to understand the life cycle of `pager` object. More detailed information can be found in `src/pager.c` source file

```
                            OPEN <------+------+
                              |         |      |
                              V         |      |
               +---------> READER-------+      |
               |              |                |
               |              V                |
               |<-------WRITER_LOCKED------> ERROR
               |              |                ^
               |              V                |
               |<------WRITER_CACHEMOD-------->|
               |              |                |
               |              V                |
               |<-------WRITER_DBMOD---------->|
               |              |                |
               |              V                |
               +<------WRITER_FINISHED-------->+


 List of state transitions and the C [function] that performs each:

   OPEN              -> READER              [sqlite3PagerSharedLock]
   READER            -> OPEN                [pager_unlock]

   READER            -> WRITER_LOCKED       [sqlite3PagerBegin]
   WRITER_LOCKED     -> WRITER_CACHEMOD     [pager_open_journal]
   WRITER_CACHEMOD   -> WRITER_DBMOD        [syncJournal]
   WRITER_DBMOD      -> WRITER_FINISHED     [sqlite3PagerCommitPhaseOne]
   WRITER_*          -> READER              [pager_end_transaction]

   WRITER_*          -> ERROR               [pager_error]
   ERROR             -> OPEN                [pager_unlock]
```

### Pager fetch page operation

Not much to note here, as this operation is pretty straight forward. Just remember to acquire a shared lock on `pager` before accessing it. In other words, remember to run `sqlite3PagerSharedLock` before using `sqlite3PagerGet`.

```c
Pager* pager;  // assume that it is initialized successfully
rc = sqlite3PagerSharedLock(pager);
if (rc != SQLITE_OK) {
  printf("Can't acquire shared lock");
  return rc;
}
DbPage *ppPage;
/**
 * as the first page (pgno starts from 1) is the database header,
 * we will start reading from index 2
 */
for(u16 pgno = 2; pgno <= noPages; pgno ++) {
  rc = sqlite3PagerGet(pager, pgno, &ppPage, 0);
  if (rc != SQLITE_OK) {
    printf("Can't read page %u", pgno);
    return rc;
  }
  printf("Number of cells of page %u: %u\n", pgno, get2byteAligned(ppPage->pData + 3));
}
```

### Pager write page operation

From the life cycle above, it is clear that, to modify the SQLite page using `pager` APIs, we have to acquire the shared lock first, and use `sqlite3PagerBegin` to start modifying the pages stored inside `pager`.

```c
Pager* pager;
DbPage *ppPage;
sqlite3PagerSharedLock(pager);
sqlite3PagerGet(pager, lastPageNo, &ppPage, 0);

/**
 * more information about `exFlag` and `subjInMemory` ([true, 0] arguments below)
 * can be found in `pager.c` - `sqlite3PagerBegin`
 */
rc = sqlite3PagerBegin(pager, true, 0);
if (rc != SQLITE_OK) {
  printf("Can't start pager txn");
  return rc;
}

// try to modify the `DbPage*` data
u16 fakeNoCell = 100;
put2byte((u8*)(ppPage->pData + 3), fakeNoCell);
rc = sqlite3PagerWrite(ppPage);
if (rc != SQLITE_OK) {
  printf("Can't modify data of last page");
  return rc;
}

// it may be useful to create a `pthread` here and try to access the last page
// from the same `pager*` object

// `DbPage*` can be flushed only when ppPage->nRef = 0
sqlite3PagerUnref(ppPage);
sqlite3PagerFlush(pager);

// clear the cache, re-fetch the page and check if the data is written correctly
sqlite3PagerClearCache(pager);
sqlite3PagerGet(pager, lastPageNo, &ppPage, 0);
assert(get2byteAligned(ppPage->pData + 3), 100);
```