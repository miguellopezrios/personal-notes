# SSTables and LSM-trees

Segments in a database represent all values written to a database in a specific period of time. These segments contain updates to keys as writes occur, but what if we sort the key-value pairs by key? This is the idea behind the SSTables.

SSTables have some advantages over log segments, such as:

- Merging multiple SSTables during compaction is simple and efficient because the keys are already sorted.
- It is not necessary to maintain an index of all the keys in memory. Due to the sorted keys, we can just maintain a sparse in-memory index that stores only a subset of the keys and points to their locations on disk.
- When receiving a read request, we need to scan a range of key-values, but we can group records into blocks and compress them before writing them to disk. Doing that, the sparse index stores the start of a compressed block, saving both disk space and I/O bandwidth.

## How can we construct and maintain SSTables?

Maintaining a sorted structure directly on disk is possible, but maintaining one in memory is much simpler and more efficient. There are several data structures, such as red-black trees or AVL trees, that allow keys to be inserted in any order while supporting to read in sorted order.

An idea of a working storage engine is the following:

- The in-memory balanced tree used by the storage engine is called **memtable**. Each write is first inserted into the memtable.

- Once the memtable is big enough, we can flush the memtable to disk as a new SSTable. The keys are already sorted, so it is a simple process. At the same time, we can continue receiving writes in the memtable.

- When receiving a read request, we first search in the memtable, then in the SSTables (starting with the most recent segment).

- In order to being able to recover from crashes, we also maintain an append-only write-ahead log (WAL) on disk. Every write is first appended to the WAL before being inserted into the memtable. If the database crashes, the memtable can be reconstructed from the WAL.

The storage engines that are based on the merging and compaction process of sorted files are called **LSM storage engines**.

We can achieve some optimizations. For example, if we receive a read request for a key that doesn't exist, we need to:

1. Search in memtable.
2. Search in all the segments of the SSTables.

This process can be optimized using a data structure called **Bloom filters**. Bloom filters can determine with certainty that a key is not present in an SSTable, allowing the database to skip reading that SSTable altogether. If the Bloom filter reports that the key may exist, the SSTable still has to be checked because Bloom filters can produce false positives but never false negatives.

# B-trees

Like SSTables, B-trees keep key-value pairs sorted by key. But their entire design is different. B-trees maintain fixed-size **blocks** or **pages**. The ideas behind the B-trees are the following:

- B-trees form a tree structure.
- The tree starts at a **root** page, and each page contains keys and references to child pages, while leaf pages contain the actual key-value pairs.
- In a page, the number of references to child pages is called the **branching factor**.

## How can we read/write keys in the B-tree?

- If we want to read a key, we start from the root page and traverse the tree until we find the leaf page containing the key.

- If we want to write a key, we need to find the page in which that key can be written. If there is enough space, it will be written. If not, the page is split, and a reference to the new page is added.

Following that algorithm, the tree remains balanced, and in a B-tree with n keys, the depth is O(log n).

## Recovering from crashes

In order to recover from crashes, B-tree implementations include a WAL. Every modification is recorded in the WAL before the corresponding B-tree pages are updated. If the database crashes, the WAL is replayed to restore any changes that had not yet been written to the tree.

## Comparing B-trees and LSM-trees

- LSM-trees are typically faster for **writes**.
- B-trees are typically faster for **reads**.
- LSM-trees normally achieve higher write throughput, mainly because they have lower write amplification (where a single write in a database results in multiple physical writes on disk).
- LSM-trees typically achieve better compression.
- A downside of LSM-trees is that compaction can interfere with reads and writes, sometimes causing performance degradation.
- An advantage of B-trees is that keys exist **exactly in one place**. This is good for databases that offer strong transactional semantics.

In summary, the choice between LSM-tree and B-tree storage engines depends on the use case.
