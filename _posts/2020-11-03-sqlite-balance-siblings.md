---
layout: post
title: SQLite balance siblings algorithm
---

## Balance siblings algorithm

### General information

Short description: [Balance deeper algorithm](https://www.sqlite.org/btreemodule.html#balance_siblings)

This function ensures that the current page and its siblings will be neither overflow nor underflow after its execution. The algorithm is quite complicated so there will be no visualization for it.

### Notes

- I won't discuss bulk load here, so the `bBulk` parameter will always be 0 in this post
- Divider cells are cells containing pointer information to child pages (aka pageno)
- From now on, when talking about `total number of cells in a page`, I am referring to the number of cells including overflow cells. If I am not, then I will refer to it clearly - such as `number of internal cells`

### Function specification

`btree.c` - `static int balance_nonroot(MemPage *pParent, int iParentIdx, u8 *aOvflSpace, int isRoot, int bBulk)`

Inputs:

- `MemPage *pParent`: The parent page of the current pages to be balanced
- `int iParentIdx`: Index of the current page in `pParent`
- `u8 *aOvflSpace`: A page-size buffer to deal with `pParent` overflow
- `int isRoot`: True if `pParent` is a root page
- `int bBulk`: True if this call is part of a bulk load

### Trigger conditions

- If the current page is not a root page
- The current `pParent` should have at most one overflow cell, and if it has, that must be the cell with `iParentIdx`. It should only happen if `sqlite3BtreeDelete` is triggered
- If the current page does not fit in the other two algorithms' trigger conditions

### Details

As this algorithm is extremely complex, I will divide it into several phases

#### Find the sibling pages to balance

- Determine the next divider cell slot - `nxDiv` - in `pParent` of the current page (including pParent's overflow cells). All up-to four divider cells that this algorithm works with will start of `nxDiv`
  - If the number of `pParent`'s cells is lower than 2, set `nxDiv = 0`
  - If the current page is the first child of `pParent`, set `nxDiv = 0`
  - If the current page is the last child of `pParent`, set `nxDiv = i-2` (at this stage, `i` is the total number of cells of `pParent`)
  - Otherwise, set `nxDiv = iParentIdx-1` - this will work with the current page all its two direct siblings

- Determine the number of cells to work with
  - `nOld = i+1;` - where `i` is either total number of cells if this number is lower than 2, or is 2 otherwise

- Find the address of right-sibling child in `pParent`, and get the pageno if it
  - `pRight = &pParent->aData[pParent->hdrOffset+8];` or
  - `pRight = findCell(pParent, i+nxDiv-pParent->nOverflow);`
  - `pgno = get4byte(pRight);`

- `i+nxDiv-pParent->nOverflow` is the expected divider slot on the left of the current cell

- Initialize all sibling pages (in descending order), and store them in `*apOld`
  - If current `pParent` is overflow, then current page - `iParentIdx` page - should be the only overflow cell. In this case, remove the overflow state and move the current cells and siblings to the workspace `apDiv`
    - `if( pParent->nOverflow && i+nxDiv==pParent->aiOvfl[0] ){ ... }`
  - Else, simply drop the current cells and siblings
    - `}else{ ... }`