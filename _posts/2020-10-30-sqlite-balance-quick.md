---
layout: post
title: SQLite Introduction to balance algorithms & balance quick algorithm
tags: [sqlite, algorithm]
---

## Introduction

There are three balance algorithms implemented in the Btree module

- Balance quick
  - `btree.c` - `static int balance_quick(MemPage *pParent, MemPage *pPage, u8 *pSpace)`
- Balance deeper
  - `btree.c` - `static int balance_deeper(MemPage *pRoot, MemPage **ppChild)`
- Balance non-root (aka balance siblings)
  - `btree.c` - `static int balance_nonroot(MemPage *pParent, int iParentIdx, u8 *aOvflSpace, int isRoot, int bBulk)`

These three algorithms will be invoked through `static int balance(BtCursor *pCur)`, which is simply a utility function to determine whether a page should be re-balanced, which balance algorithm should be used for that page, and whether that page's parent page should also be re-balanced.

If the workload is append-only, it is not worthy to spend time on Balance siblings algorithm

## Balance quick algorithm

### General information

The short description of the algorithm is written here: [Balance quick algorithm](https://www.sqlite.org/btreemodule.html#section_4_2_5_3)

This algorithm will be triggered if only the recent inserted cell is placed on the extreme right end of the
tree, i.e. the largest entry in the tree.

The algorithm will simply create a new right sibling for the current page, and move the overflow cell to this new page.

### Visualization

![Alt Visualization](https://www.sqlite.org/images/btreemodule_balance_quick.svg)

### Compile option

The algorithm can be disabled at compile-time by enabling the option `SQLITE_OMIT_QUICKBALANCE`

> gcc -DSQLITE_OMIT_QUICKBALANCE=0 application.c sqlite.c -lpthread -ldl -o output

### Function specification

`btree.c` - `static int balance_quick(MemPage *pParent, MemPage *pPage, u8 *pSpace)`

Inputs:
- `MemPage *pParent`: the parent page of the current right-most page
- `MemPage *pPage`: the current right-most page to be re-balanced
- `u8 *pSpace`: the working space to be used - this should be an empty buffer of at least 13 bytes

### Trigger conditions

- The current `pPage` is a leaf page of an intKey table
- The current `pPage` contains exactly 1 overflow cell
- The overflow cell should be inserted at the end of the current `pPage`
- The pageno of the `pParent` is not 1
- The current `pPage` is the right-most child of the `pParent` page

### Details

- A new page will be allocated, which will become the right sibling of the current page
  - `allocateBtreePage(pBt, &pNew, &pgnoNew, 0, 0)`
- Initialize a CellArray to populate the overflow cell
- Populate the new page with this CellArray
  - `rc = rebuildPage(&b, 0, 1, pNew)`
- Add a devider cell to the `pParent` page. The devider cell will be the largest entry of the `pPage`
  - `pCell = findCell(pPage, pPage->nCell-1)`
  - `insertCell(pParent, pParent->nCell, pSpace, (int)(pOut-pSpace), 0, pPage->pgno, &rc)`
- Set the right-child of the parent page to point to the new sibling page
  - `put4byte(&pParent->aData[pParent->hdrOffset+8], pgnoNew)`
- Execute the balance procedure on the parent page. This will be triggered at `int balance(BtCursor *pCur)` function