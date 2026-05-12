# Part 1: Code Changes

## Overview

For the SQLite B-Tree Split Analysis project, the SQLite source code was modified to instrument and monitor internal B-Tree behavior during insert operations.

The main goal of these modifications was to:

- Track the number of B-Tree page splits caused by SQL operations
- Observe page allocation growth before and after inserts
- Display split statistics directly through the SQLite shell interface

All build configurations and runtime experiments were performed through terminal commands during compilation and execution.

The modifications were primarily focused on adding instrumentation and debugging utilities for experimental analysis.

---

# Modified Source Files

The following SQLite source files were modified during the project:

| File | Purpose |
|---|---|
| `btree.c` | Added split counters and page tracking instrumentation |
| `shell.c.in` | Added shell-level support to display split statistics |

---

# 1. Changes in `btree.c`

File: `btree.c`

This file contains SQLite’s core B-Tree implementation. Most experimental modifications were made here. The most Important functions to focus on here are static int balance_nonroot() and the static int balance_deeper() responsible for the actual btree splits. 

## A. Adding and Incrementing Global Split Counter

A global variable was introduced at the top of the file to track the total number of B-Tree splits triggered during insert operations.

```c
// Count the total number of splits done during an insert
static int split_count = 0;
```

The global variable is incremented when a split occurs in 2 main functions static int balance_nonroot() and the static int balance_deeper() 

```c
//increment the split count when split occurs int he balancing functions
split_count ++;
```

### Purpose

This counter was used to:

- Measure how frequently page splits occur
- Compare split behavior under different workloads
- Analyze the effect of insert order and page occupancy

---

## B. Added Function Declarations

The following declarations were added at the end of the file:

```c
int get_split_count() {
  return split_count;
}

void reset_split_count() {
  split_count = 0;
}
```

### Purpose

These functions were used to:

- Retrieve the current split count
- Reset the counter between experiments

This enabled cleaner experimental runs which are easier to follow and understand.

---

# 2. Changes in `shell.c.in`

File: `shell.c.in`

The SQLite shell source was modified to expose split statistics through the command-line interface.

## A. Added Instrumentation for Page Analysis

Additional debugging logic for getting the number of pages before and after running a command were inserted specifically in the shell_exec() function to observe the following:

- Number of pages before inserts

```c
int before_pages = 0;
int after_pages = 0;
sqlite3_stmt *stmt_stats = NULL;

/* Get pages BEFORE */
sqlite3_prepare_v2(db, "PRAGMA page_count;", -1, &stmt_stats, 0);
if (sqlite3_step(stmt_stats) == SQLITE_ROW) {
  before_pages = sqlite3_column_int(stmt_stats, 0);
}
sqlite3_finalize(stmt_stats);

/* Reset split counter */
reset_split_count();
```

- Number of pages after inserts

```c
/* Get pages AFTER */
sqlite3_prepare_v2(db, "PRAGMA page_count;", -1, &stmt_stats, 0);
if (sqlite3_step(stmt_stats) == SQLITE_ROW) {
  after_pages = sqlite3_column_int(stmt_stats, 0);
}
sqlite3_finalize(stmt_stats);
```

### Purpose

This instrumentation helped analyze:

- How inserts increase page count
- Whether splits caused overflow propagation
- Storage growth patterns in SQLite B-Trees

---

## B. Shell-Level Experimental Monitoring

The shell was modified to display internal B-Tree statistics during execution.
The report is printed with the no of pages before and after along with number of splits in the command

```c
/* print report */
printf("\n=== B-TREE REPORT ===\n");
printf("Pages before: %d\n", before_pages);
printf("Pages after : %d\n", after_pages);
printf("Total splits: %d\n", get_split_count());
printf("=====================\n\n");
```

### Purpose

This allowed experiments to be conducted directly from the SQLite CLI without requiring external debugging tools.

The shell interface was used to monitor:

- Split counts
- Insert behavior
- Page growth trends

---

# Experimental Approach

The experiments focused on understanding how SQLite B-Tree nodes behave during insert-heavy workloads.

The modified system was used to:

- Run controlled insert experiments
- Observe page split frequency
- Measure database growth

All experimental executions were performed through terminal commands using this custom-built SQLite shell.
