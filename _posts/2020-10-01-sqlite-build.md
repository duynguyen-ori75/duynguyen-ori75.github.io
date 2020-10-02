---
layout: default
title: SQLite build notes
---

## Some notes about building SQLite

All modification below is done on top of SQLite version 3.10.2

### Adding custom files to SQLite source code

Assume that we have to add two files `sample.h` and `sample.c` to SQLite source code. In order to compile and pack these two with SQLite, below files should be modified:

- `Makefile.in`
  - Append `sample.h` and `sample.c` to `SRC` variable
  - Append `sample.h` to `HDR` variable
- `tool/mksqlite3c.tcl`
  - Append `sample.h` to `foreach hdr {}` (around line 95)
  - Append `sample.c` to `foreach file {}` (around line 280)

### Testing internal SQLite functions

There are several ways to test and interact with SQLite implementation, such as the traditional way - reading and modifying the test files and run them. However, it is quite complicated and time consuming, and there is another way to achieve that.

After compiling and digging into the SQLite compiled file, I found out that a lot of internal functionalities' definition come with the `SQLITE_PRIVATE` macro, which is simply `static`. This was defined in `tool/mksqlite3c.tcl` - line 84 - `# define SQLITE_PRIVATE static`. As I am not an expert in `tcl`, I decided to modify the line to `# define SQLITE_PRIVATE SQLITE_API`.

When this was done, a lot of internal functions were exposed (to the `linker`), and I was then able to `#include` the header files (eg: `btree.h`, `btreeInt.h`, `pager.h`) in my test script and compile it with `sqlite3.c` easily.

Below is my sample test script, which is compiled with this command:

```shell
gcc -I. -I../src test_script.c sqlite3.c -lpthread -ldl
```

```c
#include "sqlite3.h"
#include "btreeInt.h"

#define findCell(P,I) ((P)->aData + ((P)->maskPage & get2byteAligned(&(P)->aCellIdx[2*(I)])))

void printArr(u8* buf, int size) {
  for (int i = 0; i < size; i++) {
    if (i > 0) printf(":");
    char c = (char)(buf[i]);
    printf("%02x", c);
  }
  printf("\n");
}

int main() {
  sqlite3 *db;
  Btree* tree;
  char *err_msg;

  // open database
  int rc = sqlite3_open_v2("test.db", &db, SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE, NULL);
  if (rc) {
    printf("Can't open database %s, err: %s\n", "test.db", sqlite3_errmsg(db));
    return rc;
  }

  // open corresponding btree
  rc = sqlite3BtreeOpen(db->pVfs, "test.db", db, &tree, BTREE_SINGLE, SQLITE_ACCESS_READ);
  if (rc != SQLITE_OK) {
    printf("Can't open database %s, err: %s\n", "test.db", sqlite3_errmsg(db));
    return rc;
  }

  // sample btree metadata
  printf("Page size: %d\n", sqlite3BtreeGetPageSize(tree));

  // start read transaction -> access btree data
  sqlite3BtreeBeginTrans(tree, TRANS_READ);

  // sample query on shared btree
  printf("BtShared number of pages: %u\n", tree->pBt->nPage);

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

  /**
   * parse cell using cell offset
   */
  printf("======Read all cells using cell pointer offset=====\n");
  for(i16 offset = 0; offset < curPage->nCell * 2; offset += 2) {
    /**
     * aData also contains page header
     * as we want to read the cell only -> we have to bypass the page header area
     * the page header's size of leaf page is 8
     * -> we want to start from curPage->aData + 8
     */
    u8* cellPtr = curPage->aData + 8 + offset;
    u16 payloadOffset = get2byteAligned(cellPtr);
    printf("Payload offset of current cell: %u\n", payloadOffset);
  }
  printf("===================================================\n");

  // move cursor to specific id
  rc = sqlite3BtreeMovetoUnpacked(cur, NULL, 170, 0, &status);
  curPage = cur->apPage[cur->iPage];
  // the curPage's metadata should be different from above info
  printf("Current page number: %u\n", curPage->pgno);
  printf("Number of cells in this page: %d\n", curPage->nCell);
  printf("Current page is initialized: %u\n", curPage->isInit);
  printf("===================================================\n");

  /**
   * access root page
   */
  MemPage* rootPage = cur->apPage[0];
  printf("Is rootPage initialized: %u\n", rootPage->isInit);
  printf("rootPage's pageno: %u\n", rootPage->pgno);
  printf("rootPage's number of cells: %d\n", rootPage->nCell); // should be 2 (last pointer is not counted)
}
```

### SQLite btree structure

Will be updated later