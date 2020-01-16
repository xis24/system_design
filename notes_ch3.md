# Chapter 3: Storage and Retrieval

## Storage Engine

TWO types: log-structured and page-oriented
index data structure: speed up read queries, but every index slows down writes.

### Hash indexes

simplest indexing strategy: keep an in-memory hash map where every key is mapped to a byte offset in the data file - the location at which the value can be found. But this subject to the requirement that all the keys fit in the available RAM (since hash map is in memory). So it works well with **the value for each key is updated frequently**.

- how do we avoid eventually running out of disk space?
  break the log into segments of a certain size by closing a segment file when it reaches a certain size, making subsequent writes to a new segment file. We can perform **compaction** on these segments. Compaction means throwing away duplicate keys in the log, and keeping only the most recent update for each key.
  since compaction makes segment smaller, we also merge several segments together in the background. Segments are never modified after they have been written, so merged segments is written to a new file. Merge and compaction can be down in the background, while read and write requests as normal using old segment files. After the merge process is done, we switch read requests to using new merged segments and then old segments can be deleted.
  since each segment has its own in-memory hash table and they are relative small due to merge, lookups don't need to check many hash maps.(starting from most recent segments -> second most recent segment)

- Some of implementation issues
  - File format: binary format that first encodes the length of a string in bytes, followed by the raw string
  - Deleting record: have to append a special deletion record (tombstone). When log segments are merged, the tombstone tells the merge process to discard any previous value for the deleted key
  - Crash recovery: then in-memory hash map might be lost. So before we can store a snapshot of each segment's hash map on disk which can be loaded into memory more quickly.
  - Partially written record: include checksums, allowing such corrupted parts of the log to be detected and ignored
  - Concurrency control: as writes are appended to the log in a strictly sequential order, to have only one writer thread. Data file is append-only and immutable. So read could be done in multi-thread.

**Append only log**

- Appending and segment merging are sequential write operations that are faster than random writes.
- Concurrency and crash recovery are much simpler if segment files are append-only or immutable. For example, crash happens leave corrupted file
- Merging old segments avoids the problem of data files getting fragmented over time

#### limitation of hash table index

- hash table must fit in-memory and maintain a hash on disk and on-disk hash map is really slow
- Range queries are not efficient. For example, key like kitty00000 and kitty99999

### SSTable and LSM-Trees

SSTable (sorted string table): key value pair is sorted by key

Pro of SSTable

- Merging segments is simple and efficient, even if the files are bigger than the available memory. What if key appears in different segment? We keep the value from the most recent segment and discard the values in older segments
- in-memory **all-keys** are not needed. You only need a couple of keys to locate the requested key since they are sorted.
- Since read requests need to scan over several key-value paris in the requested range, it's possible to group those records into a block and compress it before writing it to disk

#### Constructing and maintaining SSTables

storage engine works as follows

- when a write comes in, add it to an in-memory balanced tree data structure (_memtable_).
- when memtable gets bigger than a few megabytes write it out to disk as an SSTable file. The new SSTable file becomes the most recent segment of database. While SSTable is being written out to disk, writes can continue to a new memtable instance.
- serve a read request, first try to find the key in the memtable. THen in the most recent on-disk segment, then next older segment

- run a merging and compacting process in the background to combine segment files and to discard overwritten or deleted values.

> if the database crashes, the most recent writes (in memtable but not in disk yet) are lost

we can keep a separate log on disk to which every write is immediately appended. Don't need to be sorted because it's to restore the memtable when crashes. Every time memtable is write to disk, this log can be deleted

#### Making an LSM-tree out of SSTables

SSTables and memtable is introduced in Google's Bigtable paper

LSM - tree : Log-Structured Merge-Tree by Patrick O'Neil
Lucene, indexing engine for full-text search used by Elasticsearch and Solr uses similar method for storing its term dictionary.

#### performance optimizations

LSM - tree algorithm can be slow when looking up keys that don't exist. In order to solve this issue, storage engine often use **bloom filters** (a memory-efficient data structure for approximating the contents of a set. It can tell you if a key does not appear in the db)

There are also different strategy to determine the order and timing of how SSTables are compacted and merged: size tiered and leveled compaction.
In size-tiered compaction, newer and smaller SSTables are successively merged into older and large SSTables.
In level compaction, the key range is split up into smaller SSTables and older data is moved into separate levels which allows the compaction to proceed more incrementally and use less disk space.

### B-Tree

most widely used indexing structure, standard index implementation in almost all relational database sometimes nonrelational db too.

_similarity to SSTables_: keep key-value paris sorted by key which allows efficient key-value lookups and range queries
**difference**: log-structured indexes break the db down into variable size segments, and always write sequentially. B-tree break the database down into fixed-size blocks or pages, traditionally 4KB in size, and read or write one page at a time. Its design corresponds to fixed-size disk arrangement.

each page can use an address or location which allows one page to refer to another. One page is designated as the root of the B-tree (starting point).
The number of references to child pages in one of page the B-tree is called the **branching factor**. In practice, the branching factor depending on the amount of space required to store the page references and range boundaries (several hundred)

_operation_:

- Add a new key -> find the page whose range includes the new key -> add it to that page

- update the value for existing key -> find the leaf page containing that key -> change the value -> write the page back to disk

- no free space for new key -> split into two half-full page -> parent page is updated

By this algorithm, tree remains balanced, B-tree with N-keys always has depth log(n).
4 level tree of 4KB pages with branching factor 500 can store up to 256TB

In order to prevent crash B-tree DB to leave orphan page, it's common to include an additional data structure on disk: a write-ahead log(WAL, or redo log) This is append-only file to which every B-tree modification must be written before it can be applied the page of tree itself.

Concurrency is protected by _latches_

#### B-tree optimization

- Copy-on-write replace WAL for crash recovery
- saving more space in pages by abbreviating the key

#### B-Tree vs LSM-Tree

LSM-Tree faster for writes, slow at read b/c checking different data structure and SSTables
B-Tree faster for read

##### pro of LSM-tree

- B-tree index write every data at least twice: 1. write ahead log 2. tree page itself (perhaps again when splitting)
- Log structured index also rewrite data multiple times due to repeated compaction and merging of SSTable. This effect is called write amplification.
- LSM tree are able to have higher write than B-trees b/c they have lower write amplification, and they sequentially write compact SSTable files rather than having to overwrite several pages in the tree. Sequential writes are much faster than random write.
- LSM tree can be compressed better and thus small disk usage than B-trees. B-tree leaves fragmentation when splitting.

##### con of LSM-tree

- compaction process affect performance of ongoing reads and writes.
- issue with compaction arises at high write throughput: disk write bandwidth needs to be shared between initial write and the compaction thread in the background. If compaction is not well configured, it can happen compaction can't keep up with incoming writes.
- pro of B-tree: key exists in exactly one place in the index, but log structured storage engine may have multiple copies of the same key in different segment.
- B-tree provides consistent performance. In new data store, log-structured indexes are becoming popular.
