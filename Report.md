## Introduction to B-Trees
A B-tree is a self-balancing tree data structure designed to store sorted data and support efficient search, insertion, and deletion operations. Unlike traditional binary search trees, B-trees are optimized for systems where data is stored on disk (such as databases and file systems). They achieve high efficiency by reducing the number of disk accesses required during operations.
### Why do we need B-trees
When working with large datasets that cannot fit entirely in memory, disk access becomes the primary bottleneck. B-trees address this issue through:
High branching factor (fanout), Balanced structure, Block-oriented design
This makes B-trees significantly more efficient than binary trees for storage systems.
### Structure of a Node in B-tree
Each B-tree node contains:
- A list of sorted keys
- A list of child pointers (one more than the number of keys)
- A flag indicating whether it is a leaf node
This structure allows efficient navigation and storage of large datasets.
### Key Properties of B-trees
A B-tree of order m (or minimum degree t) follows these rules:
- Each node can have at most m children.
- Each internal node (except root) has at least ⌈m/2⌉ children.
- The root has at least two children if it is not a leaf.
- All leaves are at the same depth (perfectly balanced).
- A node with k children contains k−1 keys, stored in sorted order.
- Subtrees maintain strict key ordering between parent keys.
Using minimum degree t:
- Each node has between t−1 and 2t−1 keys.
- Each node has at most 2t children.
### Basic Operations
1) Search
2) Insertion
3) Deletion

In this project/report, the focus will be on understanding and analyzing the insertion operation and B-tree split operations in depth, particularly how they are implemented in SQLite. We will explore the internal working of node splitting, propagation of splits, and how SQLite manages B-tree structures at a low level, including code-level insights and implementation details. This will involve reverse engineering the behavior of B-tree splits within SQLite to gain a comprehensive understanding of their role in maintaining balance and efficiency in database systems.


## B-Tree Splits
A B-tree split is an operation that occurs when a node becomes full and cannot accommodate a new key during insertion. Instead of allowing the node to overflow, the B-tree maintains its structure by:
- Dividing the full node into two smaller nodes
- Moving the middle key (median) up to the parent node
- Redistributing the remaining keys between the two nodes
This process ensures that the tree remains balanced and continues to satisfy all B-tree properties.
Splits are essential for maintaining the efficiency and correctness of a B-tree:
- Prevent Overflow: Nodes have a maximum capacity. Splitting ensures this limit is never exceeded.
- Maintain Balance: All leaves must remain at the same level. Splits help preserve this property.
- Ensure Logarithmic Performance: By controlling tree height, splits guarantee efficient operations 𝑂(log 𝑛)
- Enable Growth: Splits allow the tree to grow dynamically without degrading performance.
Without splits, insertions would eventually break the structure of the tree, leading to inefficient searches and imbalance.
B-tree splits are a core mechanism that allows the tree to grow while staying balanced and efficient. They ensure that no node exceeds its capacity and that the structure remains optimal for disk-based storage systems.
### Basic Idea of how Splitting works
1) When inserting a key into a B-Tree we locate the leaf node where the key should be inserted using a normal search operation B-Tree
2) there a 2 ways to splits
    - Pre-Split : where we split the nodes if they are full before inserting while we travel down the tree
    - Post-Split : where we first insert the key intot he node and then check if it full or not and split if it is full, splitting propagates upward if nessesary 
3) During the Actual split we identify the middle key the node, and take it up to the parent and take all the smaller keys to the left child node and the remaining keys to the right child



## Execution Pipeline of B tree Splits

### INSERT query parsed → insert.c
#### **Step 1: SQL Parsing and Insert Invocation**
When an `INSERT` statement is issued, the SQLite SQL parser processes the query and translates it into an internal representation. As part of this parsing phase, the function **`sqlite3Insert()`** is invoked.
This function serves as the central entry point for handling all `INSERT` operations in SQLite. It is responsible for orchestrating the entire insertion workflow, including table access, data preparation, constraint handling, and ultimately writing the data into the database.

#### **Step 2: Initialization and Table Setup**
Within the `sqlite3Insert()` function, several preparatory steps are executed to establish the necessary environment for the insertion:
* **Opening the Target Table**:
  The function **`sqlite3OpenTable()`** is called to open a cursor on the target table. This cursor provides access to the underlying B-tree structure where the data will be stored.
* **Handling Auto-Increment Columns**:
  If the table contains an `AUTOINCREMENT` primary key, the function **`autoIncBegin()`** is invoked. This ensures that the appropriate row ID is generated and maintained consistently, preventing reuse of previously deleted row IDs.
These steps ensure that the database is in a valid and ready state before any data manipulation occurs.

#### **Step 3: Data Preparation and Transformation**
Before inserting the data into the table, SQLite performs several transformations to ensure correctness and consistency:
* **Loading Values into Registers**:
  The values specified in the `INSERT` statement are first loaded into internal memory registers used by the SQLite Virtual Database Engine (VDBE). These registers act as temporary storage for row data during execution.
* **Applying Column Affinity**:
  The function **`sqlite3TableAffinity()`** is used to enforce the declared data types (affinities) of the table columns. This step ensures that values are converted to the appropriate storage class (e.g., INTEGER, TEXT) as required by the schema.
* **Computing Generated Columns**:
  If the table contains generated (computed) columns, the function **`sqlite3ComputeGeneratedColumns()`** is called. This computes the values of such columns based on their defined expressions before the row is inserted.
These transformations guarantee that the data conforms to the schema and is correctly formatted for storage.


### VDBE insert opcodes are generated
#### **Step 4: Row Construction**
Once all input values have been prepared and validated, SQLite constructs a complete database record representing the row to be inserted.
* **Record Assembly using `OP_MakeRecord`**:
  The Virtual Database Engine (VDBE) generates the opcode **`OP_MakeRecord`** to combine the individual column values stored in registers into a single contiguous record format.
* **Record Format and Encoding**:
  This operation serializes the row data into SQLite’s internal record format, which includes:
  * A header section containing metadata such as column types and sizes.
  * A payload section containing the actual column values.
* **Purpose of This Step**:
  The constructed record is the exact binary representation that will be stored in the table’s B-tree. This ensures efficient storage and retrieval within the database engine.

#### **Step 5: Record Insertion into B-Tree**
After the row has been constructed, SQLite proceeds to insert it into the underlying table structure.
* **Insertion using `OP_Insert`**:
  The VDBE generates the opcode **`OP_Insert`**, which is responsible for inserting the constructed record into the table’s B-tree.
* **Key Responsibilities of `OP_Insert`**:
  * Inserts the record at the correct location based on the row ID (or primary key for WITHOUT ROWID tables).
  * Interacts directly with the B-tree layer to place the record into the appropriate leaf node.
  * Maintains the sorted order of records within the B-tree.
* **Interaction with Storage Engine**:
  During execution, `OP_Insert` invokes lower-level B-tree functions (such as `sqlite3BtreeInsert()`), which handle:
  * Locating the correct page
  * Writing the record into the page
  * Triggering page splits if the target page does not have sufficient space
* **Outcome**:
  At the end of this step, we have a OP_Insert bytecode that when executed internally calls the sqlite3BtreeInsert() function  in btree.c.


### VDBE executes sqlite3BtreeInsert() in btree.c
#### **Step 6: B-Tree Insertion Entry Point (`sqlite3BtreeInsert`)**
The function **`sqlite3BtreeInsert()`** serves as the primary entry point for all record insertions into SQLite’s B-tree storage layer. It is invoked by higher-level components (such as the VDBE through the `OP_Insert` opcode) to perform the actual physical insertion of a record into the database.
**Core Responsibilities**
* **Validation of Cursor State**:
  The function first verifies that the B-tree cursor is in a valid state for insertion. This includes ensuring that the cursor is correctly positioned and associated with the appropriate table or index.
* **Transaction and Lock Management**:
  It ensures that a write transaction is active and that the necessary locks are acquired. This guarantees data consistency and prevents concurrent write conflicts during the insertion process.
* **Preparation for Insertion**:
  Before performing the insert, the function prepares the required data structures, including formatting the key and payload that will be stored in the B-tree node.

**Key Functional Operations**
* **Cursor Positioning**:
  The cursor is positioned at the correct location within the B-tree where the new record should be inserted. This ensures that the sorted order of keys is maintained.
* **Cell Creation**:
  The record is converted into a B-tree cell format, which includes:
  * The key (e.g., row ID or index key)
  * The associated payload (record data)
* **Insertion into Page**:
  The newly created cell is inserted into the appropriate leaf page of the B-tree. If sufficient space is available, the insertion is completed directly.
* **Triggering Tree Balancing (if required)**:
  If the target page does not have enough free space to accommodate the new cell, the function initiates the B-tree balancing process. This may involve:
  * Splitting the page
  * Redistributing cells
  * Updating parent nodes

**Outcome**
Upon successful execution, the record is inserted into the correct location within the B-tree structure, preserving the sorted order and structural properties of the tree. If necessary, balancing operations ensure that the tree remains efficient for future queries and insertions.

### Leaf node located
#### **Step 7: Cursor Positioning within the B-Tree**
Before inserting a new record, SQLite must determine the exact location within the B-tree where the record should be placed. This is achieved through cursor positioning functions.
**Key Functions**
* **`btreeMoveto()`**
* **`sqlite3BtreeIndexMoveto()`**
**Purpose and Functionality**
* **Locating the Target Leaf Page**:
  These functions traverse the B-tree structure starting from the root node down to the appropriate leaf node. The traversal is guided by key comparisons to ensure that the correct insertion point is identified.
* **Positioning the Cursor**:
  Once the correct leaf node is located, the cursor is positioned precisely at the location where the new record should be inserted. This position may either point to:
  * An existing key (in case of updates or conflicts), or
  * The appropriate gap between keys for a new insertion
**Significance**
* **Leaf Node Requirement**:
  In SQLite’s B-tree implementation, all insertions occur strictly at the **leaf level**. Internal nodes are used only for navigation and do not store actual table records.
* **Maintaining Sorted Order**:
  Proper cursor positioning ensures that the B-tree maintains its sorted structure, which is critical for efficient search, insertion, and deletion operations.

#### **Step 8: Cell Creation and Formatting**
Once the correct insertion point has been identified, SQLite prepares the data for storage in the B-tree by converting it into a structured format known as a *cell*.
**Key Function**
* **`fillInCell()`**
**Purpose and Functionality**
* **Conversion to B-tree Cell Format**:
  The function **`fillInCell()`** transforms the prepared row data into a B-tree cell. Each cell represents a single entry in a B-tree page and contains:
  * The **key** (e.g., row ID or index key)
  * The **payload** (serialized row data)
* **Payload Management**:
  The function determines how much of the payload can be stored directly within the B-tree page and how much must be stored externally.
* **Handling Overflow Pages**:
  If the payload size exceeds the available space within the page, the excess data is stored in **overflow pages**, and the cell maintains references to these pages.
**Significance**
* **Storage Abstraction**:
  SQLite does not store rows as raw records within pages. Instead, all data is encapsulated in **cells**, which provide a consistent and efficient storage structure.
* **Efficient Space Utilization**:
  By splitting large payloads between the main page and overflow pages, SQLite ensures optimal use of page space while still supporting large records.
* **Foundation for B-tree Operations**:
  Proper cell construction is essential for insertion, deletion, and balancing operations, as all B-tree manipulations operate on cells rather than raw row data.

These steps collectively ensure that the record is correctly positioned and formatted before being inserted into the B-tree structure, maintaining both data integrity and structural efficiency.
 

### If space → insert directly
#### **Step 9: Insertion of Cell into B-Tree Page**
After the cell has been constructed, SQLite proceeds to insert it into the appropriate B-tree page in memory. This step is responsible for physically placing the record within the page structure.
**Key Functions**
* **`insertCellFast()`**
* **`insertCell()`**
**Purpose and Functionality**
* **Placing the Cell into Page Memory**:
  These functions handle the insertion of the newly created cell into the target leaf page. The cell is inserted at the correct position within the page to preserve the sorted order of keys.
* **Updating the Cell Pointer Array**:
  Each B-tree page maintains a **cell pointer array**, which stores offsets to individual cells within the page. Upon insertion:
  * A new pointer is added to this array
  * Existing pointers may be shifted to maintain sorted order
* **Managing Free Space within the Page**:
  The functions update the page’s internal free space metadata, including:
  * Adjusting the available free space
  * Allocating space for the new cell
  * Updating fragmentation details if necessary
**Optimized vs General Insertion**
* **`insertCellFast()`**:
  This is an optimized path used when conditions allow for a straightforward insertion (e.g., sufficient contiguous free space is available). It minimizes overhead and improves performance.
* **`insertCell()`**:
  This is the general-purpose insertion function used when additional handling is required, such as managing fragmented free space or performing more complex memory adjustments.
**Significance**
* **Efficient In-Memory Update**:
  This step ensures that the cell is correctly integrated into the page structure without immediately requiring disk I/O.
* **Preservation of B-Tree Properties**:
  By maintaining the correct ordering and structure within the page, SQLite ensures that subsequent operations (search, insert, delete) remain efficient.
**Key Observation**
* **Insertion Completes if Space is Available**:
  If the target page has sufficient free space to accommodate the new cell, the insertion process concludes at this stage without requiring further structural changes.
* **Btree Split is striigerd if the page is full**:
  If the target page does not have sufficient free space to accommodate the new cell, the function initiates the B-tree balancing process. This involves:
  * Splitting the page
  * Redistributing cells
  * Updating parent nodes

This step represents the final stage of a simple insertion. More complex scenarios, such as insufficient space, lead to additional operations like page splitting and tree balancing.

### If full → balance() triggered
#### **Step 10: Overflow Detection and Split Trigger**
During the insertion process, after attempting to place a cell into a B-tree page, SQLite checks whether the page has exceeded its storage capacity.
**Code Condition**

```c
if( pPage->nOverflow ){
    rc = balance(pCur);
}
```

### **Purpose and Functionality**
* **Detection of Page Overflow**:
  The variable `pPage->nOverflow` indicates that one or more cells could not be accommodated within the available space of the page.
* **Triggering Rebalancing**:
  When overflow is detected, SQLite invokes the **`balance()`** function to resolve the space constraint.

### **Significance**
* **Indication of Full Page**:
  This condition signifies that the current page is full and cannot accommodate additional records.
* **Initiation of Structural Changes**:
  Instead of failing the insertion, SQLite dynamically reorganizes the B-tree structure to make space, ensuring continued efficient operation.

## **Step 11: Page Balancing and Split Execution (`balance()`)**

The function **`balance()`** is the core mechanism responsible for maintaining the structural integrity of the B-tree during insertions that cause overflow.


### **High-Level Responsibilities**
* Redistributes cells across one or more pages
* Splits pages when necessary
* Updates parent nodes to reflect structural changes

### **Internal Variants of Balancing**

#### **1. `balance_quick()` (Optimized Case)**
* **Usage Condition**:
  Invoked when insertion occurs at the **rightmost edge** of the B-tree.
* **Behavior**:
  Instead of performing a full redistribution, SQLite:
  * Allocates a new page
  * Moves the overflowing cell directly into the new page
* **Advantage**:
  Minimizes computational overhead by avoiding full-page rebalancing.

#### **2. `balance_nonroot()` (General Case)**
* **Usage Condition**:
  Applied when the overflow occurs in a **non-root page**, which is the most common scenario.
* **Behavior**:
  * Considers the current page along with its sibling pages
  * Collects all cells from these pages
  * Redistributes them evenly across available pages
* **Outcome**:
  Ensures balanced space utilization and maintains optimal B-tree structure.

#### **3. `balance_deeper()` (Root Split Case)**
* **Usage Condition**:
  Triggered when the **root page** itself overflows.
* **Behavior**:
  * Creates a new root node
  * Splits the existing root into child pages
  * Increases the height of the B-tree
* **Significance**:
  This is the only scenario in which the B-tree grows in height.

## **Step 12: Cell Redistribution**

Within the balancing process, SQLite performs redistribution of cells across pages.

### **Key Operations**
* **Collection of Cells**:
  All relevant cells from the current page and its siblings are gathered into temporary structures (e.g., arrays such as `b.apCell[]`).
* **Even Redistribution**:
  The collected cells are redistributed across pages in a way that:
  * Maintains sorted order
  * Balances space utilization
  * Minimizes fragmentation
### **Significance**
* **Core Split Logic**:
  This step represents the actual implementation of the page split.
* **Structural Optimization**:
  Proper redistribution ensures that no page remains overly full or underutilized, preserving B-tree efficiency.

## **Step 13: Page Reconstruction (`rebuildPage()`)**

After redistribution, each affected page is reconstructed to reflect the new layout.

### **Key Function**
* **`rebuildPage()`**

### **Purpose and Functionality**
* **Rebuilding Page Structure**:
  The function reconstructs each page by:
  * Writing the page header
  * Rebuilding the cell pointer array
  * Organizing the cell content area
* **Ensuring Consistency**:
  It ensures that all structural components of the page are correctly aligned and updated after redistribution.

### **Significance**
* **Maintains Page Integrity**:
  Guarantees that the page remains in a valid format for future operations.
* **Prepares for Persistent Storage**:
  The reconstructed page is now ready for eventual writing to disk.

## **Step 14: Parent Node Update**

Following page splitting and reconstruction, SQLite updates the parent node to reflect the changes in the B-tree structure.

### **Purpose and Functionality**
* **Insertion of Divider Key**:
  A separator (divider key) is inserted into the parent node to distinguish between the newly formed child pages.
* **Updating Child Pointers**:
  The parent node’s pointers are updated to reference the correct child pages resulting from the split.

### **Significance**
* **Maintaining Tree Connectivity**:
  Ensures that navigation from parent to child nodes remains correct.
* **Preserving B-tree Properties**:
  The hierarchical structure and ordering guarantees of the B-tree are maintained.

## **Overall Outcome**

These steps collectively handle scenarios where a simple insertion is not possible due to space constraints. Through overflow detection, balancing, redistribution, and structural updates, SQLite ensures that:

* The B-tree remains balanced
* Data is stored efficiently
* Performance of search and insert operations is preserved

### Pager writes pages to disk safely
## **Step 15: Final State and Persistence**

After the completion of the balancing process, the insertion operation reaches its final state.

### **Outcome**
* **B-Tree Integrity Maintained**:
  The tree structure is fully balanced and satisfies all B-tree properties.
* **Cursor State Reset**:
  The cursor is repositioned appropriately, ensuring consistency for subsequent operations.
* **Insertion Completion**:
  The new record is successfully integrated into the database structure.
* **Persistent Storage via Pager**:
  The pager subsystem ensures that all modified pages are safely written to disk, maintaining durability and consistency of the database.


### Conculusion
The SQLite insertion workflow demonstrates a highly structured and efficient pipeline that transforms a high-level SQL INSERT statement into a low-level modification of the B-tree storage structure. Beginning with SQL parsing and progressing through data preparation, record construction, and cursor positioning, the system ensures that all input data conforms to the schema and is correctly formatted for storage.

At the storage layer, the B-tree insertion process handles both straightforward and complex scenarios. When sufficient space is available, records are directly inserted into the appropriate leaf page. In cases of overflow, SQLite dynamically invokes its balancing mechanism, which redistributes cells, performs page splits, and updates parent nodes to maintain structural integrity. Special handling for root splits ensures that the tree can grow in height while preserving its properties.

Overall, this workflow highlights SQLite’s robust design, where correctness, efficiency, and consistency are maintained through careful coordination between the parser, virtual machine, B-tree layer, and pager subsystem. The result is a reliable insertion mechanism that scales gracefully with increasing data size while ensuring optimal performance and data integrity.

