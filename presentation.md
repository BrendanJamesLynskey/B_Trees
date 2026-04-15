# B-Trees

**Computer Science Fundamentals Series**

B-tree operations · B+ trees · Disk I/O · Database indexes · Node splitting · LSM comparison

*Mid-level software engineer track -- 20 slides*

---

## Table of Contents

1. [Motivation -- Disk-Based Storage](#slide-02--motivation--disk-based-storage)
2. [B-Tree Definition](#slide-03--b-tree-definition)
3. [B-Tree Node Structure](#slide-04--b-tree-node-structure)
4. [B-Tree Search](#slide-05--b-tree-search)
5. [B-Tree Insertion](#slide-06--b-tree-insertion)
6. [Node Splitting in Detail](#slide-07--node-splitting-in-detail)
7. [B-Tree Deletion](#slide-08--b-tree-deletion)
8. [Deletion -- Merging & Borrowing](#slide-09--deletion--merging--borrowing)
9. [B+ Trees](#slide-10--b-trees-1)
10. [B+ Tree Range Queries](#slide-11--b-tree-range-queries)
11. [B* Trees](#slide-12--b-trees-2)
12. [B-Tree Height Analysis](#slide-13--b-tree-height-analysis)
13. [Disk I/O Model & Why Fanout Matters](#slide-14--disk-io-model--why-fanout-matters)
14. [B-Trees in Databases](#slide-15--b-trees-in-databases)
15. [B-Trees in File Systems](#slide-16--b-trees-in-file-systems)
16. [LSM-Trees vs B-Trees](#slide-17--lsm-trees-vs-b-trees)
17. [Fractal Trees & Write-Optimised Structures](#slide-18--fractal-trees--write-optimised-structures)
18. [Practical Considerations](#slide-19--practical-considerations)
19. [Summary & Further Reading](#slide-20--summary--further-reading)

---

## Slide 02 -- Motivation -- Disk-Based Storage

### The memory-disk gap

Main memory is fast but limited. Databases must store far more data than fits in RAM, so most data lives on disk (SSD or HDD).

- Random disk read: ~100 us (SSD) or ~10 ms (HDD)
- Random RAM access: ~100 ns -- three to five orders of magnitude faster
- Sequential disk reads are much cheaper than random -- amortise seek cost over large blocks
- OS reads data in fixed-size pages (typically 4 KB); databases use 8--16 KB pages

### Minimising I/O

The dominant cost in disk-based data structures is the number of disk page reads/writes. An ideal index structure:

- Keeps the tree **short** so search touches few pages
- Packs many keys per node so each page read yields maximum information
- Maintains balance to guarantee worst-case performance
- Exploits sequential access patterns where possible

> **Design goal:** minimise the number of disk I/Os per operation -- not CPU comparisons. This is the fundamental insight behind B-trees.

---

## Slide 03 -- B-Tree Definition

### Formal definition (Knuth, 1973)

A B-tree of order *m* (also called minimum degree *t* where *m = 2t*) satisfies:

| Property | Description |
|----------|------------|
| **Root** | Has between 1 and *m - 1* keys (may have as few as 0 children if it is a leaf) |
| **Internal nodes** | Each non-root internal node has between *ceil(m/2)* and *m* children |
| **Keys per node** | Each node with *k* children stores exactly *k - 1* keys |
| **Sorted keys** | Keys within each node are in ascending order |
| **Balanced** | All leaves appear on the same level |
| **Subtree ordering** | For a key *K_i*, all keys in the left subtree are < *K_i* and all keys in the right subtree are > *K_i* |

### Why these constraints?

- **Minimum fill** guarantees nodes are at least half full -- good space utilisation
- **Balance** guarantees O(log n) height -- predictable search cost
- **High fanout** keeps the tree shallow -- fewer disk reads per search

> A B-tree of order 1000 with 1 billion keys has height <= 3. Three disk reads to find any key among a billion.

---

## Slide 04 -- B-Tree Node Structure

### Anatomy of a B-tree node

Each node occupies one disk page and contains:

```
+---+---+---+---+---+---+---+---+---+---+---+---+---+
| P0| K1| P1| K2| P2| K3| P3| ...       | Kn| Pn|
+---+---+---+---+---+---+---+---+---+---+---+---+---+

Ki = key i (sorted)
Pi = pointer to child i (disk page address)
```

- **Keys** are stored in sorted order within the node
- **Pointers** (child page addresses) interleave with keys
- Leaf nodes have no child pointers (or null pointers)
- Each node may also store associated data values (in a B-tree) or just keys (in a B+ tree)

### Typical sizing

| Parameter | Typical Value |
|-----------|--------------|
| Page size | 8 KB -- 16 KB |
| Key size | 8 -- 64 bytes |
| Pointer size | 6 -- 8 bytes |
| Keys per node | 100 -- 500+ |
| Fanout | 101 -- 501+ |

> High fanout is the key advantage. A binary tree with 1 million keys has height ~20. A B-tree with fanout 500 has height ~3.

---

## Slide 05 -- B-Tree Search

### Algorithm

```
BTREE-SEARCH(node, key):
    i = 0
    while i < node.num_keys and key > node.keys[i]:
        i += 1
    if i < node.num_keys and key == node.keys[i]:
        return (node, i)          # found
    if node.is_leaf:
        return NOT_FOUND
    DISK-READ(node.children[i])   # one I/O
    return BTREE-SEARCH(node.children[i], key)
```

### Cost analysis

- Each level of the tree requires one disk read
- Within a node, binary search over keys: O(log m) comparisons (CPU-bound, cheap)
- Total disk I/Os: O(log_t n) where t is the minimum degree
- Total CPU comparisons: O(log_t n * log m) = O(log n)

### Example: searching for key 42

```
         [30 | 60]                   <- Root (1 disk read)
        /    |    \
  [10|20] [35|42|55] [70|80|90]     <- Level 1 (1 disk read)

  Read root -> 42 > 30, 42 < 60 -> follow middle pointer
  Read node -> find 42 at position 1 -> done
  Total: 2 disk reads
```

---

## Slide 06 -- B-Tree Insertion

### Algorithm overview

1. Search for the correct leaf node
2. If the leaf has room, insert the key in sorted position -- done
3. If the leaf is full (m - 1 keys), **split** the node

### Proactive (preemptive) splitting

Instead of splitting bottom-up after overflow, split any full node encountered on the way **down** during the search. This guarantees:

- At most one split per level
- The parent always has room for the promoted key
- Single-pass insertion -- no need to backtrack up the tree

### Insertion example (order 5, max 4 keys per node)

```
Insert 25 into:  [10 | 20 | 30 | 40]   <- full leaf

Step 1: Split at median (30)

        [30]                 <- median promoted to parent
       /    \
  [10|20]  [30|40]           <- two half-full nodes

Step 2: Insert 25 into left child -> [10|20|25]
```

> Proactive splitting is the standard approach used in practice. It avoids the complexity of bottom-up split propagation and simplifies concurrent access.

---

## Slide 07 -- Node Splitting in Detail

### Split algorithm

```
SPLIT-CHILD(parent, i):
    full_node = parent.children[i]     # has m-1 keys
    median = full_node.keys[t-1]       # middle key
    new_node = allocate new page

    # Right half of keys/children go to new_node
    new_node.keys = full_node.keys[t .. m-1]
    new_node.children = full_node.children[t .. m]

    # Left half stays in full_node
    full_node.num_keys = t - 1

    # Insert median into parent
    insert median into parent.keys at position i
    parent.children[i+1] = new_node

    DISK-WRITE(full_node)
    DISK-WRITE(new_node)
    DISK-WRITE(parent)
```

### Split cost

| Operation | Count |
|-----------|-------|
| Disk reads (search path) | O(log_t n) |
| Disk writes (split) | 3 per split (old node, new node, parent) |
| Splits per insertion | at most O(log_t n) in worst case |
| Amortised splits | O(1) per insertion |

> In practice, splits are rare. Most insertions just add a key to a non-full leaf -- one disk write.

---

## Slide 08 -- B-Tree Deletion

### Three cases

| Case | Situation | Action |
|------|-----------|--------|
| **1** | Key found in a leaf node | Remove the key directly |
| **2** | Key found in an internal node | Replace with predecessor (or successor) from a leaf, then delete from leaf |
| **3** | Key not in current node | Ensure child has >= t keys before recursing (fix underflow proactively) |

### Fixing underflow before recursion

When descending to a child that has only *t - 1* keys (minimum), we must ensure it has at least *t* keys before entering it:

- **Borrow from sibling:** if an adjacent sibling has >= t keys, rotate a key through the parent
- **Merge:** if both siblings have exactly t - 1 keys, merge the child with a sibling and pull the separating key down from the parent

> Like insertion, deletion uses **proactive fixing** -- we ensure each node on the path down has enough keys so we never need to backtrack.

---

## Slide 09 -- Deletion -- Merging & Borrowing

### Borrowing from a sibling

```
Before (child has t-1 keys, right sibling has >= t keys):

  Parent:   [...| 30 |...]
              /       \
  Child: [10|20]    [35|40|50|60]   <- right sibling has spare keys

After rotation:

  Parent:   [...| 35 |...]
              /       \
  Child: [10|20|30]  [40|50|60]     <- borrowed 35, demoted 30
```

### Merging two nodes

```
Before (both child and sibling have t-1 keys):

  Parent:   [...| 30 |...]
              /       \
  Child: [10|20]    [35|40]

After merge:

  Parent:   [...]               <- 30 pulled down
              |
  Merged: [10|20|30|35|40]     <- combined node
```

> If the parent was the root and had only one key, the merged node becomes the new root -- the tree shrinks in height. This is the only way a B-tree gets shorter.

---

## Slide 10 -- B+ Trees

### Key difference from B-trees

In a B+ tree, **all data records live in the leaf nodes**. Internal nodes store only keys and child pointers -- they are a pure index.

| Feature | B-Tree | B+ Tree |
|---------|--------|---------|
| Data location | Any node | Leaves only |
| Leaf linking | None | Doubly-linked list |
| Internal node keys | Separate data | Copies/separators only |
| Fanout | Lower (data in internal nodes) | Higher (index-only internal nodes) |
| Range queries | Requires tree traversal | Follow leaf pointers |
| Point query | May terminate at internal node | Always reaches a leaf |

### Leaf-level linked list

```
Internal:     [30 | 60]
             /    |    \
Leaves:  [10|20] <-> [30|35|42] <-> [60|70|80]
          data         data           data

Each leaf points to next and previous leaf.
Sequential scan = follow the chain.
```

> B+ trees are the default index structure in virtually all relational databases. The leaf-level linked list makes range queries and full scans dramatically faster.

---

## Slide 11 -- B+ Tree Range Queries

### Algorithm

```
RANGE-QUERY(tree, low, high):
    leaf = search for leaf containing low
    results = []
    while leaf is not null:
        for each (key, value) in leaf:
            if key > high:
                return results
            if key >= low:
                results.append((key, value))
        leaf = leaf.next_leaf      # follow linked list
    return results
```

### Cost analysis

| Operation | Cost |
|-----------|------|
| Find starting leaf | O(log_t n) disk reads |
| Scan matching leaves | O(k / b) disk reads, where k = result size, b = keys per leaf |
| Total | O(log_t n + k/b) |

### Why this matters

```sql
SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-03-31'
ORDER BY order_date;
```

- B+ tree on `order_date`: find the first matching leaf, then scan sequentially
- The leaf linked list provides the data **already sorted** -- no additional sort step
- Adjacent leaves are often physically adjacent on disk -- sequential I/O

> This is why `ORDER BY` on an indexed column is essentially free when the query uses that index.

---

## Slide 12 -- B* Trees

### Stricter fill guarantee

A B* tree is a variant where every non-root node must be at least **2/3 full** (compared to 1/2 in a standard B-tree).

| Property | B-Tree | B* Tree |
|----------|--------|---------|
| Minimum fill | 1/2 | 2/3 |
| Split strategy | Split one node into two | Redistribute between siblings, then split two nodes into three |
| Space utilisation | ~50--69% typical | ~66--81% typical |
| Average fanout | Higher than B-tree | Even higher |

### Two-to-three split

When a node overflows and its sibling is also full:

```
Before:  [A B C D] [E F G H]     <- both full (order 5)

Instead of splitting one into two,
redistribute across three nodes:

After:   [A B C] [D E] [F G H]   <- 2 nodes become 3
         (separator keys promoted to parent)
```

### Benefits

- Better space utilisation -- fewer disk pages needed for the same data
- Higher effective fanout -- shallower trees
- Fewer splits overall

> B* trees are used in some file systems (HFS+ historically used B* trees for its catalog file).

---

## Slide 13 -- B-Tree Height Analysis

### Height bound

For a B-tree of order *m* (minimum degree *t = ceil(m/2)*) with *n* keys:

```
Height h <= log_t((n + 1) / 2)
```

### Concrete examples

| Keys (n) | Order (m) | Min degree (t) | Max height | Disk reads per search |
|----------|-----------|-----------------|-----------|----------------------|
| 1,000 | 100 | 50 | 2 | 2 |
| 1,000,000 | 100 | 50 | 4 | 4 |
| 1,000,000,000 | 100 | 50 | 6 | 6 |
| 1,000,000 | 500 | 250 | 3 | 3 |
| 1,000,000,000 | 500 | 250 | 4 | 4 |
| 1,000,000,000 | 1000 | 500 | 4 | 4 |

### Why B-trees beat binary trees

A balanced BST with 1 billion keys: height ~30 (30 disk reads).
A B-tree with order 1000 and 1 billion keys: height ~3--4 (3--4 disk reads).

> The root node is almost always cached in memory. So in practice, a search in a billion-key B-tree often requires only **2--3 disk reads**.

---

## Slide 14 -- Disk I/O Model & Why Fanout Matters

### The external memory model

In the external memory (or I/O) model:

- Memory holds *M* elements
- Disk transfers data in blocks of *B* elements
- Cost metric: number of block transfers (I/Os)

### Fanout and tree height

```
Fanout f = (page size) / (key size + pointer size)

Example: 16 KB page, 8-byte keys, 8-byte pointers
f = 16384 / 16 = 1024

Height for n = 10^9:
h = ceil(log_1024(10^9)) = ceil(3.0) = 3
```

### I/O costs comparison

| Structure | Search | Insert | Range (k results) |
|-----------|--------|--------|--------------------|
| Sorted array | O(log_B n) | O(n / B) | O(log_B n + k/B) |
| Binary search tree | O(log n) | O(log n) | O(log n + k) |
| B-tree / B+ tree | O(log_B n) | O(log_B n) | O(log_B n + k/B) |
| Hash table | O(1) avg | O(1) avg | O(n / B) -- no range support |

> B-trees achieve the optimal I/O complexity for comparison-based search. The only structure with better *point* query I/O is hashing -- but hashing cannot support range queries or ordering.

---

## Slide 15 -- B-Trees in Databases

### InnoDB (MySQL)

- Default storage engine since MySQL 5.5
- **Clustered index:** the primary key B+ tree stores the actual row data in its leaves -- the table *is* the index
- Secondary indexes store the primary key value in their leaves; a secondary index lookup requires two B+ tree traversals (index + clustered)
- Page size: 16 KB default; typical fanout: ~500--1200

### PostgreSQL

- Heap-organised tables (not clustered by default)
- B-tree is the default index type (`CREATE INDEX` uses B-tree)
- Uses Lehman-Yao B-trees with right-link pointers for lock-free concurrent reads
- Page size: 8 KB; typical fanout: ~200--500
- Supports `INCLUDE` columns for covering indexes (index-only scans)

### Other databases

| Database | B-Tree variant | Notes |
|----------|---------------|-------|
| SQLite | B+ tree | Both table and index structures |
| Oracle | B+ tree | Index-organised tables (IOT) option |
| SQL Server | B+ tree | Clustered and non-clustered indexes |
| MongoDB (WiredTiger) | B+ tree | Default storage engine since 3.2 |

---

## Slide 16 -- B-Trees in File Systems

### NTFS (Windows)

- Master File Table (MFT) uses B+ trees for directory indexing
- File names sorted in B+ tree leaves for fast directory lookups
- Large directories with thousands of entries remain fast

### HFS+ (macOS, legacy)

- Catalog file is a B* tree mapping file IDs to metadata
- Extents overflow file is also a B* tree
- Replaced by APFS in 2017 (which uses different structures)

### Btrfs (Linux)

- "B-tree file system" -- name is literal
- Copy-on-write B-trees throughout: file extents, directory entries, checksums, snapshots
- Multiple B-trees share a common root tree
- Copy-on-write enables atomic snapshots and transactions

### Other file systems using B-trees

| File System | B-Tree usage |
|-------------|-------------|
| XFS | B+ trees for directory entries, free space, inodes, extents |
| ReiserFS | B+ tree (S+ tree) for all metadata and small file data |
| ZFS | Uses DMU with indirect block trees (not classical B-trees) |
| ext4 | HTree (hash-indexed B-tree variant) for directories |

---

## Slide 17 -- LSM-Trees vs B-Trees

### Log-Structured Merge Trees

LSM-trees buffer writes in an in-memory *memtable*, then flush to immutable sorted runs (SSTables) on disk. Periodic compaction merges runs.

| Property | B-Tree | LSM-Tree |
|----------|--------|----------|
| Write pattern | Random (update page in place) | Sequential (append + compact) |
| Write amplification | Lower for updates | Higher (compaction rewrites data) |
| Write throughput | Moderate | High (sequential I/O) |
| Read latency | Predictable O(log_B n) | May check multiple levels |
| Space amplification | ~50--69% node utilisation | Multiple copies during compaction |
| Range queries | Excellent (B+ tree leaf scan) | Good (after compaction) |
| Concurrency | Page-level locks or MVCC | Lock-free writes to memtable |

### When to prefer each

- **B-trees:** read-heavy OLTP, predictable latency, range scans, general-purpose workloads
- **LSM-trees:** write-heavy workloads, time-series, logging, append-mostly data

### Engines using LSM-trees

`RocksDB` | `LevelDB` | `Cassandra` | `ScyllaDB` | `HBase` | `CockroachDB` (on top of RocksDB/Pebble)

> Many modern databases let you choose: WiredTiger (MongoDB) supports both. TiDB uses RocksDB (LSM) for storage and B+ tree indexes logically.

---

## Slide 18 -- Fractal Trees & Write-Optimised Structures

### The write amplification problem

B-trees write an entire page for each modified key. If a page holds 500 keys but only one changed, you still write the full page. Write amplification = (bytes written to disk) / (bytes of actual data written).

### Fractal trees (Buffered B-trees)

Each internal node has a **message buffer**. Writes are inserted into the buffer at the root and lazily pushed down during reads or when buffers fill.

```
         [30 | 60]  buffer: [ins(45), del(12), ins(72)]
        /    |    \
    [...]  [...]  [...]   <- messages trickle down over time
```

| Property | B-Tree | Fractal Tree |
|----------|--------|-------------|
| Point query | O(log_B n) | O(log_B n) |
| Insert | O(log_B n) | O(log_B n / B^e) for 0 < e < 1 |
| Range query | O(log_B n + k/B) | O(log_B n + k/B) |
| Write amplification | Higher | Significantly lower |

### Other write-optimised structures

| Structure | Key idea |
|-----------|----------|
| **LSM-tree** | Batch writes, sequential flush, periodic compaction |
| **Bw-tree** | Append delta records to pages; consolidate lazily (used in SQL Server Hekaton) |
| **LA-tree** | Lazy-adaptive tree; combines B-tree and LSM ideas |

> TokuDB (now archived) used fractal trees. The ideas live on in PerconaFT and influenced modern hybrid designs.

---

## Slide 19 -- Practical Considerations

### Choosing page size

- Larger pages = higher fanout = shallower tree = fewer I/Os per search
- Larger pages = more wasted space if nodes are sparse = higher write amplification
- SSD random I/O is fast enough that 4--16 KB pages work well
- HDD benefits more from larger pages (amortise seek latency)

### Bulk loading

Instead of inserting keys one by one, **bulk loading** builds a B-tree bottom-up:

1. Sort all keys
2. Fill leaf pages sequentially (100% full)
3. Build internal pages from leaf-level separators
4. Result: compact, fully packed tree -- one sequential write pass

> `CREATE INDEX CONCURRENTLY` in PostgreSQL and InnoDB's `ALTER TABLE ... ADD INDEX` use optimised bulk-loading strategies.

### Concurrency control

| Approach | Description |
|----------|------------|
| **Lock coupling** | Lock parent, lock child, release parent. Top-down, prevents deadlocks. |
| **B-link trees** | Right-link pointers allow readers to move right without locks even during splits (Lehman-Yao). |
| **Optimistic** | Read without locks; validate before write. Works well for read-heavy workloads. |
| **OLFIT** | Optimistic Lock-Free Index Traversal -- version counters on nodes detect concurrent modifications. |

---

## Slide 20 -- Summary & Further Reading

### Key takeaways

- B-trees are the dominant on-disk index structure because they minimise disk I/O through high fanout and guaranteed balance
- A B-tree with order 1000 can index a billion keys in 3--4 levels -- 3--4 disk reads per search
- B+ trees keep all data in leaves with a linked list, making range queries and sequential scans efficient
- Proactive splitting (insertion) and proactive fix-up (deletion) enable single-pass, top-down algorithms
- B-trees power the indexes in virtually every relational database and many file systems
- LSM-trees trade read performance for write throughput; fractal trees reduce write amplification while preserving B-tree query performance
- The right choice depends on workload: read-heavy favours B-trees, write-heavy may favour LSM-trees

### Recommended reading

| Source | Description |
|--------|------------|
| **Cormen et al.** | *Introduction to Algorithms* (CLRS) -- Chapter 18: B-Trees |
| **Graefe** | "Modern B-Tree Techniques" -- comprehensive 2011 survey |
| **Kleppmann** | *Designing Data-Intensive Applications* -- Chapter 3: Storage and Retrieval |
| **Bayer & McCreight** | "Organization and Maintenance of Large Ordered Indexes" (1970) -- the original B-tree paper |
| **Lehman & Yao** | "Efficient Locking for Concurrent Operations on B-Trees" (1981) -- B-link trees |
| **CMU 15-445** | Andy Pavlo's Database Systems course -- storage engines and indexing lectures |
