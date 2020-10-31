---
layout: post
title: SQLite balance deeper algorithm
---

## Balance deeper algorithm

### General information

The short description: [Balance deeper algorithm](https://www.sqlite.org/btreemodule.html#section_4_2_5_1)

This algorithm will be triggered if only the root page of the current Btree is overfull. The root page's content will be moved to a newly allocated child page, and this child will then be the only child of the root page. Visualization below:

![Alt Visualization](https://www.sqlite.org/images/btreemodule_balance_deeper.svg)

### Function specification

`btree.c` - `static int balance_deeper(MemPage *pRoot, MemPage **ppChild)`

Inputs:
- `MemPage *pRoot`: the current root page of the Btree
- `MemPage **ppChild`: the `MemPage` representation of the new child page will be written to this pointer

### Trigger conditions

- The current `pPage` is the root page
- The current `pRoot` is overflowed
- There is no valid `pCur` except the current balancing one

### Details

- Mark the root page as writeable
  - `sqlite3PagerWrite(pRoot->pDbPage)`
- Allocate a new page - `pChild`, and copy the content of root page to this new page
  - `allocateBtreePage(pBt,&pChild,&pgnoChild,pRoot->pgno,0)`
  - `copyNodeContent(pRoot, pChild, &rc)`
- Copy the overflow cells from `pRoot` to `pChild`
  - `memcpy(pChild->aiOvfl, pRoot->aiOvfl, pRoot->nOverflow*sizeof(pRoot->aiOvfl[0]))`
  - `memcpy(pChild->apOvfl, pRoot->apOvfl, pRoot->nOverflow*sizeof(pRoot->apOvfl[0]))`
  - `pChild->nOverflow = pRoot->nOverflow`
- Clean the content of `pRoot`, then install `pChild` as the right-child
  - `zeroPage(pRoot, pChild->aData[0] & ~PTF_LEAF)`
  - `put4byte(&pRoot->aData[pRoot->hdrOffset+8], pgnoChild)`