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
  -- File format
