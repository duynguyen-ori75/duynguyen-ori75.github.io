---
layout: post
title: SQLite balance siblings algorithm - 2nd part
---

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

#### Allocate k new pages and try to reuse old pages where possible

