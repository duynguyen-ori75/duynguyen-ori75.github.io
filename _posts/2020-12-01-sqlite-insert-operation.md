---
layout: post
title: SQLite insert operation
---

This is done with SQLite version 3.7.11

## Reason why this post is made

Usually, an insert operation on a B-tree data structure will be straight-forward and easy-to-understand.

For SQLite, the story is more complicated, because the implementation of the insert operator should be production-ready, and its level of detail is high.

Furthermore, its API is quite unfriendly to newcomers (like me), which prevents them from testing it in a script-like manner

```c
int sqlite3BtreeInsert(
  BtCursor *pCur,                /* Insert data into the table of this cursor */
  const void *pKey, i64 nKey,    /* The key of the new record */
  const void *pData, int nData,  /* The data of the new record */
  int nZero,                     /* Number of extra 0 bytes to append to data */
  int appendBias,                /* True if this is likely an append */
  int seekResult                 /* Result of prior MovetoUnpacked() call */
)
```

Therefore, I write this post to help not only me but the others who are also new to SQLite, to understand the whole flow of `sqlite3BtreeInsert`, and how to test our custom modification

To enable custom modification, please check this post: [Notes about compiling SQLite from source](2020/10/01/sqlite-build.html)

### Quick analysis of the API

```c
int sqlite3BtreeInsert(
  BtCursor *pCur,                /* Insert data into the table of this cursor */
  const void *pKey, i64 nKey,    /* The key of the new record */
  const void *pData, int nData,  /* The data of the new record */
  int nZero,                     /* Number of extra 0 bytes to append to data */
  int appendBias,                /* True if this is likely an append */
  int seekResult                 /* Result of prior MovetoUnpacked() call */
)
```

Let's assume that we are inserting a new record into an INTKEY table, and then analyze the parameters:

- `BtCursor *pCur`: the `BtCursor*` is only used to determine the `MemPage*` (which points to the correct disk image of the page) that a new cell should be inserted into
- `i64 nKey`: the primary key of the new record
- `const void *pData, int nData`: the raw data of the cell, stored in `char*` format. This is the conclusion I made when I took a quick look at `vdbe.c` - `OP_InsertInt`.

## Implementation details

## How to test SQLite insert operator

We should do it similarly to how we did in this post: [SQLite B-tree module](2020/10/02/sqlite-btree.html)

For SQLite insert operator, it is required to know the cell content in bytes beforehand. Therefore, it is recommended to test with a table whose schema is as simple as possible, so it would be easy to write a test script

## Sample test script