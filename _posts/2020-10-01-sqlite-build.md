---
layout: post
title: Notes about compiling SQLite from source
---

All modification below is done on top of SQLite version 3.10.2

## Adding custom files to SQLite source code

Assume that we have to add two files `sample.h` and `sample.c` to SQLite source code. In order to compile and pack these two with SQLite, below files should be modified:

- `Makefile.in`
  - Append `sample.h` and `sample.c` to `SRC` variable
  - Append `sample.h` to `HDR` variable
- `tool/mksqlite3c.tcl`
  - Append `sample.h` to `foreach hdr {}` (around line 95)
  - Append `sample.c` to `foreach file {}` (around line 280)

## Testing internal SQLite functions

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

  sqlite3BtreeCommit(tree);
}
```