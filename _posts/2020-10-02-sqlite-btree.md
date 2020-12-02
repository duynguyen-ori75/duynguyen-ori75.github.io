---
layout: post
title: SQLite btree module
tags: [sqlite, algorithm]
---

## SQLite architecture

![SQLite architecture](/assets/btree-arc.jpg)

There are three main modules in the storage (aka. backend) of SQLite: `Btree`, `Pager` and `OS interface`. This post will focus on the `Btree` module.

## Btree module

In SQLite, the `Btree` module contains several structs. Amongst those, below are the most notable ones:

- `struct Btree`
- `struct BtShared`
- `struct BtCursor`
- `struct MemPage`
- `struct CellInfo`

### Btree, BtShared and BtCursor

`struct Btree` is simply a representative of the internal btree for a single connection to the database. In a process where multiple SQLite connections are declared, there is no `Btree*` object that is concurrently used by more than one connection.

`struct BtShared` is the object represents a single database file. In multi-thread SQLite build, multiple `Btree*` objects can share a single `BtShared*` object. This is known as [`Shared-cache`](https://sqlite.org/sharedcache.html). If `Shared-cache` is disabled, all `Btree*` connection will have their own separated `BtShared*` object.

A `BtCursor*` object is simply a pointer to an `entry` (aka. `tuple` or `cell`) in `BtShared*`'s leaf nodes. `BtCursor*` object will contain information about the current `MemPage*` and `CellInfo*`, which helps us get data in the `Btree`.

### MemPage and CellInfo

`struct MemPage` is an in-memory instance representing database pages. An `MemPage*` object contains decoded information from the a raw data page, as well as a pointer which points to loaded in-memory raw data. The format of raw data is below:


```shell
#      |----------------|
#      | page header    |   8 bytes for leaf nodes.  12 bytes for interior nodes
#      |----------------|
#      | cell pointer   |   |  the 2-byte payload offset to the corresponding cell - 2 bytes per cell.
#      | array          |   |  all offsets will be stored in descending order
#      |                |   v  which means that the payload of the first entry will be stored at the end of the page
#      |----------------|
#      | unallocated    |
#      | (free) space   |
#      |----------------|
#      | cell payload   |   ^  grows upwards
#      | area           |   |  the corresponding payload of all stored tuples
#      |----------------|
```

Whenever a `MemPage*` is initialized (`MemPage*->isInit`), we can access any `cell` data using `MemPage*->xParseCell`. This function will output a `CellInfo` information, which contains data and metadata about any `cell`

### Sample code

```c
// Global helper functions
#define findCell(P,I) ((P)->aData + ((P)->maskPage & get2byteAligned(&(P)->aCellIdx[2*(I)])))

void printArr(u8* buf, int size) {
  for (int i = 0; i < size; i++) {
    if (i > 0) printf(":");
    char c = (char)(buf[i]);
    printf("%02x", c);
  }
  printf("\n");
}
```

```c
// inside int main()
// define a cursor to query the btree
BtCursor *cur = (BtCursor*)malloc(sqlite3BtreeCursorSize());
sqlite3BtreeCursorZero(cur);
sqlite3BtreeCursor(tree, 2, 0, 0, cur);

// move the cursor to the first entry of the table
int status;
rc = sqlite3BtreeFirst(cur, &status);
if (rc != SQLITE_OK) {
  printf("Can't move to first entry, err: %s\n", sqlite3_errmsg(db));
  return rc;
}
printf("Move status: %d\n", status);

/**
  * access content of first page
  * it should be an internal page (without payload)
  * therefore, the size of payload should be 0
  */
printf("Current page index in cur->apPage: %i\n", cur->iPage);
// cur->iPage should not be 0 - as cur will point to a leaf page
MemPage *curPage = cur->apPage[cur->iPage];
printf("Current page number: %u\n", curPage->pgno);
printf("Current page is initialized: %u\n", curPage->isInit);
printf("Size of payload: %u\n", curPage->noPayload);

/**
  * parse cell and show cell data
  */
printf("Number of cells: %d\n", curPage->nCell);
CellInfo cInfo;
printf("==========Read all cells using cell index=========\n");
for(u8 idx = 0; idx < curPage->nCell; idx ++) {
  i64 payloadOffset = get2byteAligned(curPage->aCellIdx + 2*idx);
  printf("-------------Cell idx %u----------------\n", idx);
  printf("Payload offset of cell %u: %lli\n", idx, payloadOffset);
  curPage->xParseCell(curPage, findCell(curPage, idx), &cInfo);
  /**
    * nKey should be equal to idx
    * we're trying to read the cell data from curPage
    * with idx is the index of the cell we're reading
    * and cInfo.nKey is the stored key of that cell
    */
  printf("Cell info - nKey: %llu\n", cInfo.nKey);
  printf("Cell info - payload size: %u\n", cInfo.nPayload);
  printf("Cell info - content size: %u\n", cInfo.nSize);
  /**
    * read local payload of current cell. Two ways:
    * - read cInfo.nLocal bytes from cInfo.pPayload
    * - read cInfo.nLocal bytes from curPage->aData + payloadOffset + 2
    *   as cell's content contains two info: intKey(2 bytes) and payload,
    *   therefore, we have to read from payloadOffset + 2
    */
  u8 payload1[cInfo.nLocal], payload2[cInfo.nLocal];
  // first method
  memcpy(&payload1, cInfo.pPayload, cInfo.nLocal);
  // second method
  memcpy(&payload2, curPage->aData + payloadOffset + 2, cInfo.nLocal);
  // two results should be the same
  int comp = memcmp(&payload1, &payload2, cInfo.nLocal);
  printf("Data comparison result: %d\n", comp);
  // debug payload
  // printArr(payload1, cInfo.nLocal);
  // printArr(payload2, cInfo.nLocal);
}
printf("===================================================\n");
```