# Chapter 3: Storage and Retrieval

## Storage Engine

two types: log-structured and page-oriented
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
- when memtable gets bigger than a few megebytes write it out to disk as an SSTable file. The new SSTable file becomes the most recent segment of database. While SSTable is being written out to disk, writes can continue to a new memtable instance.
- serve a read request, first try to find the key in the memtable. THen in the most recent on-disk segment, then next older segment

- run a merging and compacting process in the background to combine segment files and to discard overwritten or deleted values.

> if the databse crashes, the most recent writes (in memtable but not in disk yet) are lost

we can keep a separate log on disk to which every write is immediately appended. Don't need to be sorted because it's to restore the memtable when crashes. Every time memtable is write to disk, this log can be deleted

#### Making an LSM-tree out of SSTables

SSTables and memtable is introduced in Google's Bigtable paper

LSM - tree : Log-Structured Merge-Tree by Patrick O'Neil
Lucene, indexing engine for full-text search used by Elasticsearch and Solr uses similar method for storing its term dictionary.

#### performance optimizations

LSM - tree algo can be slow when looking up keys that don't exist. In order to solve this issue, storage engine often use **bloom filters** (a memory-efficient data structure for approximating the contents of a set. It can tell you if a key does not appear in the db)

There are also different strategy to determine the order and timing of how SSTables are compacted and merged: size tiered and leveled compaction.
In size-tiered compaction, newer and smaller SSTables are successively merged into older and large SSTables.
In level compaction, the key range is split up into smaller SSTables and older data is moved into separate levels which allows the compaction to proceed more incrementally and use less disk space.
