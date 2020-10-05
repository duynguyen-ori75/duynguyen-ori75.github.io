---
layout: post
title: SQLite shared-cache mode
---

SQLite Shared cache mode can be managed by using `SQLITE_OPEN_PRIVATECACHE` or `SQLITE_OPEN_SHAREDCACHE` flag when opening new SQLite connection.

```c
sqlite3 *db2;
char *err_msg;
int rc = sqlite3_open_v2("test.db", &db2, SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_PRIVATECACHE, NULL);
```

By creating a thread and create a separated connection on that, we can observe the behavior of this flag

### Sample code

```
#include "sqlite3.h"
#include "btreeInt.h"
#include <pthread.h>

//const int flags = SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_SHAREDCACHE;
const int flags = SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_PRIVATECACHE;

void *threadFunc(void *arg) {
  sqlite3 *db2;
  char *err_msg;

  sqlite3_open_v2("test.db", &db2, flags, NULL);

  printf("DB2 - BtShared address: %p\n", db2->aDb->pBt->pBt);
}

int main() {
  sqlite3 *db;
  char *err_msg;

  sqlite3_open_v2("test.db", &db, flags, NULL);
  printf("DB1 - BtShared address: %p\n", db->aDb->pBt->pBt);

  pthread_t threadId;
  pthread_create(&threadId, NULL, &threadFunc, NULL);
  pthread_join(threadId, NULL);

  return 0;
}
```

### Result

When `flags`'s `SQLITE_OPEN_SHAREDCACHE` bit is set, the result `stdout` is:

```shell
DB1 - BtShared address: 0x55c3c2e09b08
DB2 - BtShared address: 0x55c3c2e09b08
```

When `SQLITE_OPEN_PRIVATECACHE` is enabled, the following is printed:

```shell
DB1 - BtShared address: 0x5586e5d408e8
DB2 - BtShared address: 0x7f24780011a8
```