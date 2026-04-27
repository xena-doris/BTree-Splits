# Failure Analysis: B-Tree Splits in SQLite (with Source Code Mapping)

## Overview

SQLite uses a **B-tree data structure** implemented primarily in `btree.c`, with durability and atomicity handled by `pager.c` and `wal.c`.

* B-tree logic: `src/btree.c`
* Page management: `src/pager.c`
* WAL logging: `src/wal.c`

B-tree splits are triggered during insert operations when a page overflows and must be rebalanced.

# 1. What happens when data size increases significantly?

## 1.1 Increased Tree Height

### Code Reference

* Root split logic:

  * `balance_deeper()` — `src/btree.c`

```c
static int balance_deeper(MemPage *pRoot){
    /* Create new child page and push root content down */
}
```

### Explanation

* When root overflows, a new root is created.
* Tree height increases.

### Impact

* More page accesses per lookup
* Increased disk I/O latency

## 1.2 Frequent Page Splits

### Code Reference

* Insert path:

  * `sqlite3BtreeInsert()` → `insertCell()` → `balance_nonroot()`
* File: `src/btree.c`

```c
if( pPage->nFree < nCell ){
    rc = balance_nonroot(pParent, iIdx, &pPage);
}
```

### Explanation

* Triggered when page has insufficient space.

### Impact

* Rebalancing overhead
* CPU + disk cost increases

## 1.3 Fragmentation and Space Inefficiency

### Code Reference

* Free space tracking:

  * `MemPage.nFree`
  * `allocateSpace()` — `src/btree.c`

```c
int allocateSpace(MemPage *pPage, int nByte, int *pIdx){
    /* Finds fragmented free space inside page */
}
```

### Explanation

* Fragmented pages reduce effective capacity.

### Impact

* Premature splits
* Increased page count

## 1.4 Overflow Pages Expansion

### Code Reference

* Overflow handling:

  * `insertCell()` — `src/btree.c`

```c
if( nPayload > pPage->maxLocal ){
    /* Allocate overflow pages */
}
```

* Traversal:

  * `accessPayload()`

```c
while( ovflPgno ){
    rc = getOverflowPage(pBt, ovflPgno, &pOvfl, &next);
}
```

### Explanation

* Large records stored across multiple pages.

### Impact

* More complex split operations
* Higher risk during failure

## 1.5 Write Amplification

### Code Reference

* Page modification:

  * `sqlite3PagerWrite()` — `src/pager.c`

```c
int sqlite3PagerWrite(DbPage *pDbPage){
    /* Mark page dirty and journal original */
}
```

### Explanation

* Each split modifies:

  * Child page
  * New sibling page
  * Parent page

### Impact

* Multiple disk writes per insert

# 2. What happens if a component fails?

## 2.1 Crash During Split Operation

### Code Reference

* Split logic:

  * `balance_nonroot()` — `src/btree.c`

```c
/* Redistribute cells among sibling pages */
```

### Risk Point

* Between:

  * Writing child page
  * Updating parent pointer

### Mitigation

#### Journal Write

* `pager_write()` — `src/pager.c`

```c
rc = sqlite3OsWrite(pPager->jfd, pData, pageSize, offset);
```

#### Rollback

* `pagerPlaybackRollback()`

```c
/* Restore pages from journal */
```

## 2.2 Disk Write Failure (I/O Error)

### Code Reference

* Low-level write:

  * `sqlite3OsWrite()` — VFS layer

```c
rc = sqlite3OsWrite(pPager->fd, pData, nByte, offset);
```

### Risk

* Partial write (torn page)

### Mitigation

#### WAL Mode

* `sqlite3WalFrames()` — `src/wal.c`

```c
/* Append changes to WAL instead of DB file */
```

## 2.3 Memory Allocation Failure

### Code Reference

* Page allocation:

  * `allocateBtreePage()` — `src/btree.c`

```c
if( pPage==0 ){
    return SQLITE_NOMEM;
}
```

* Memory allocation:

```c
pNew = sqlite3MallocZero(sz);
if( pNew==0 ) return SQLITE_NOMEM;
```

### Behavior

* Operation aborted safely

## 2.4 Concurrency and Locking Issues

### Code Reference

* Transaction start:

  * `sqlite3BtreeBeginTrans()` — `src/btree.c`

```c
rc = sqlite3PagerBegin(pBt->pPager, wrflag>1, 0);
```

* Lock handling:

  * `pagerLockDb()` — `src/pager.c`

### Behavior

* Single writer enforced
* Errors:

```c
return SQLITE_BUSY;
```

## 2.5 Overflow Page Chain Corruption

### Code Reference

* Overflow traversal:

  * `getOverflowPage()` — `src/btree.c`

```c
rc = getOverflowPage(pBt, pgno, &pPage, &nextPgno);
```

### Risk

* Broken chain → unreadable data

### Detection

* `checkTreePage()` — integrity check

```c
/* Verify page structure and pointers */
```

# 3. What assumptions does this system rely on?

## 3.1 Atomic Page Writes

### 📍 Code Reference

* Commit phase:

  * `sqlite3PagerCommitPhaseOne()` — `src/pager.c`

```c
rc = sqlite3OsWrite(pPager->fd, pData, pageSize, offset);
```

### Assumption

* Entire page written atomically

## 3.2 Reliable Storage Medium

### Code Reference

* Implicit assumption across pager layer

* Error handling:

```c
if( rc!=SQLITE_OK ) return rc;
```

### Risk

* Silent corruption not always detectable immediately

## 3.3 B-Tree Invariants Always Hold

### Code Reference

* Rebalancing:

  * `balance_nonroot()`

```c
/* Ensure keys remain sorted and pages balanced */
```

### Validation

* `checkTreePage()`

```c
/* Validate ordering and structure */
```

## 3.4 Availability of Free Pages

### Code Reference

* Page allocation:

  * `allocateBtreePage()`

```c
rc = allocateBtreePage(pBt, &pNew, &pgno, ...);
```

### Risk

* If allocation fails → insert fails

## 3.5 Correct Parent-Child Relationships

### Code Reference

* Parent update inside split:

  * `balance_nonroot()`

```c
/* Update parent cell pointers */
```

### Risk

* Broken links → unreachable subtree

## 3.6 Single Writer Assumption

### Code Reference

* Lock enforcement:

  * `sqlite3PagerBegin()`

```c
if( pPager->eLock>=RESERVED_LOCK ){
    return SQLITE_BUSY;
}
```

# Summary

| Failure Scenario     | Code Location                  | Protection       |
| -------------------- | ------------------------------ | ---------------- |
| Split crash          | `balance_nonroot()`            | Journal / WAL    |
| Disk write failure   | `sqlite3OsWrite()`             | Rollback         |
| Memory failure       | `allocateBtreePage()`          | Abort safely     |
| Concurrency issue    | `sqlite3BtreeBeginTrans()`     | Locking          |
| Overflow corruption  | `getOverflowPage()`            | Integrity checks |
| Structural violation | `balance_nonroot()`            | Validation       |
| Non-atomic writes    | `sqlite3PagerCommitPhaseOne()` | WAL              |

# 💡 Final Insight

> The correctness of B-tree splits in SQLite depends less on the split algorithm itself and more on the **pager subsystem (`pager.c`)**, which guarantees atomicity, durability, and recovery.

Without journaling:

* Even a single crash during `balance_nonroot()` could corrupt the entire database.