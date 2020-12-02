---
layout: post
title: SQLite balance siblings algorithm - 2nd part
tags: [sqlite, algorithm]
---

First post can be found here: [Balance sibling 1st part](2020/11/03/sqlite-balance-siblings-1.html)

## Balance siblings algorithm (cont.)

> Note: the details below are from SQLite 3.7.17

### Details (cont.)

#### Determine the number of pages needed to hold all `nCell` cells

- Determine the number of bytes of space available on each sibling
  - `usableSpace = pBt->usableSize - 12 + leafCorrection;`
- Compute spaced used on the i-th sibling page (`szNew[i]`) and index in `apCell[]` and `szCell[]` for the first cell to the right of the i-th sibling page
  - The computation will be executed inside `for(subtotal=k=i=0; i<nCell; i++){ ... }` loop
    - `szNew[k] = subtotal - szCell[i];`
    - `cntNew[k] = i;`
- After the above execution block, the total number of sibling pages will be stored in `k`
- The cell packing is biased toward the siblings on the left side

#### Balance the packing of siblings

- Loop through all siblings from the right to the left, and compare two neighbors
  - Inside `for(i=k-1; i>0; i--){ ... }` loop
    - Right sibling - `int szRight = szNew[i];`
    - Left sibling - `int szLeft = szNew[i-1];`
    - Try to move cells from left sibl to right sibl until `szRight > 0` or `szRight+szCell[d]+2 > szLeft-(szCell[r]+2)`
      - `r` is the index of right-most cell in left sibling
      - `d` is the index of first cell to the left of right sibling

#### Allocate k new pages, free un-usable pages and sort them

- Loop through all pages - inside `for(i=0; i<k; i++){ ... }` loop
  - If the current page is reusable - `if( i<nOld ){ ... }`, simply mark it writable
    - `rc = sqlite3PagerWrite(pNew->pDbPage);`
  - Else, allocate a new page
    - `rc = allocateBtreePage(pBt, &pNew, &pgno, (bBulk ? 1 : pgno), 0);`
- All the `MemPage *pNew;` will be stored in `MemPage *apNew[NB+2];` array
- Free any old pages that were not reused as new pages
  - `while( i<nOld ){ ... }` loop
-  Sort the new pages in accending order - a tiny optimization
  - `for(i=0; i<k-1; i++){ ... }`
    - `for(j=i+1; j<k; j++){ ... }`
  - Insertion sort is used in this case, as the number of new pages is very small

#### Distribute the data in apCell[] across the new pages evenly

- Loop through all new pages - inside `for(i=0; i<nNew; i++){ ... }` loop
  - Assemble `MemPage *pNew` with `&apCell[j]` using `cntNew[i]` and `&szCell[j]`
    - `j` is the pointer to the expected cell
    - cells from `apCell[j]` -> `apCell[cntNew[i]]` will be placed in `MemPage *pNew`
  - If the recently-assembled page is not the right-most sibling, insert a divider cell into `pParent`
    - `insertCell(pParent, nxDiv, pCell, sz, pTemp, pNew->pgno, &rc);`

#### Balance shallower

- If `pParent` is the root page, and the content of `pParent` is empty, execute the `balance shallow` algorithm
- It is simply copying the content from the right-most child into the `pParent` page, and decrease the level of B-tree