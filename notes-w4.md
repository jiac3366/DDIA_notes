
## Chapter3

- ### SSTables and LSM-Trees
    
    - Base on the storing a log of key-value pairs in a CSV-like format, indexed with an inmemory hash map, do some change: key-value pairs is sorted by key, We call this format Sorted String Table, or SSTable for short. SSTables have several big advantages over log segments with
hash indexes:

        - Easily merge segments. we can keep the value from the most recent segment and discard the values in older segments

        - Index becomes more sparse(Just like AVL). No longer need to keep an index
of all the keys in memory because key is orderly stored. eg: We can find B if we see A or C.

        - The data between the sparse in-memory index can be compressed, Besides saving disk space, compression also reduces the I/O bandwidth use.

    - We can choose red-black trees or AVL trees this kind of sorted structure as the index of the SSTables. How to maintain them (Tree index and SSTables)? 

        - For write:

            - A write request come, just add it to index.

            - when index gets bigger than one threshold, write it out to disk as an SSTable file. This can be done efficiently because the
tree already maintains the key-value pairs sorted by key.The new SSTable file
becomes the most recent segment of the database.

        - For read:

            - In order to serve a read request, first try to find the key in the memtable, then in
the most recent on-disk segment, then in the next-older segment, etc        

        - From time to time, run a merging and compaction process in the background to
combine segment files and to discard overwritten or deleted values

    - If db crash, in-memory index will lost which leads to loss data. We can keep a separate log on disk to
which every write is immediately appended, That log
is not in sorted order, but that doesn’t matter, because its only purpose is to restore
the memtable after a crash.Every time the memtable is written out to an SSTable, the
corresponding log can be discarded.


- ### Making an LSM-tree out of SSTables

    - The algorithm described here is essentially what is used in LevelDB and RocksDB, key-value storage engine libraries that are designed to be embedded into other
applications. Among other things, LevelDB can be used in Riak as an alternative to
Bitcask. Similar storage engines are used in Cassandra and HBase.

    - Storage engines that are based on this principle of
merging and compacting sorted files are often called LSM storage engines. ps: Lucene, an indexing engine for full-text search used by Elasticsearch and Solr, uses a
similar method for storing its term dictionary. It maps from term to postings list is kept in
SSTable-like sorted files, which are merged in the background as needed 

    - Performance optimizations:

        - Bloom filters: the LSM-tree algorithm can be slow when looking up keys that do not
exist in the database: you have to check the memtable, then the segments all the way
back to the oldest (possibly having to read from disk for each one) before you can be
sure that the key does not exist.  Bloom filter can tell you if a key does not
appear in the database, and thus saves many unnecessary disk reads for nonexistent
keys.

        - compacted and merged strategies (size-tiered and leveled
compaction). LevelDB and RocksDB use leveled compaction (hence the name of Lev‐
elDB), HBase uses size-tiered, and Cassandra supports both.

    - Advantage of LSM-Tree: 

        -  efficiently perform range queriesand since data is stored in sorted order
        
        - the LSM-tree can support remarkably high write throughput because the disk
writes are sequential 


- ### B-Tree

    - The log-structured indexes we saw earlier break the database down into variable-size
segments, typically several megabytes or more in size, and always write a segment
sequentially

    - By contrast, B-trees break the database down into fixed-size blocks or
pages, traditionally 4 KB in size (sometimes bigger), and read or write one page at a
time. This design corresponds more closely to the underlying hardware, as disks are
also arranged in fixed-size blocks

    - The number of references to child pages in one page of the B-tree is called the
branching factor. For example, in Figure 3-6 the branching factor is six. In practice,
the branching factor depends on the amount of space required to store the page refer‐
ences and the range boundaries, but typically it is several hundred.

    - If you want to add a new key, you
need to find the page whose range encompasses the new key and add it to that page.
If there isn’t enough free space in the page to accommodate the new key, it is split
into two half-full pages, and the parent page is updated to account for the new subdi‐
vision of key ranges（Figure 3-7）

    -  Most databases can fit into a B-tree that is three or four levels
deep, so you don’t need to follow many page references to find the page you are look‐
ing for. (A four-level tree of 4 KB pages with a branching factor of 500 can store up to
256 TB.)

    - Making B-trees reliable

        -  some write operations require several different pages to be overwritten. For example, if you split a page because an insertion caused it to be overfull, you need to
write the two pages that were split, and also overwrite their parent page to update the
references to the two child pages. This is a dangerous operation, because if the data‐
base crashes after only some of the pages have been written, you end up with a corrupted index (e.g., there may be an orphan page that is not a child of any parent). In order to make the database resilient to crashes, it is common for B-tree implemen‐
tations to include an additional data structure on disk: a write-ahead log (WAL, also
known as a redo log). This is an append-only file to which every B-tree modification
must be written before it can be applied to the pages of the tree itself. When the data‐
base comes back up after a crash, this log is used to restore the B-tree back to a consistent state.

        - careful concurrency
control is required, This is typically done
by protecting the tree’s data structures with latches (lightweight locks)

    - B-tree optimizations

        - 1.Instead of overwriting pages and maintaining a WAL for crash recovery, some
databases (like LMDB) use a copy-on-write scheme [21]. A modified page is
written to a different location, and a new version of the parent pages in the tree is
created, pointing at the new location. Also useful for concurrency control.

        - 2.We can save space in pages by not storing the entire key, but abbreviating it.
Especially in pages on the interior of the tree, keys only need to provide enough
information to act as boundaries between key ranges. Packing more keys into a
page allows the tree to have a higher branching factor, and thus fewer levels. (B+ Tree)

        - 3. It is better to query on LSM-trees where needs to scan over a
large part of the key range in sorted order. Because In general, pages can be positioned anywhere on disk; there is nothing requiring
pages with nearby key ranges to be nearby on disk. If a query needs to scan over a
large part of the key range in sorted order, that page-by-page layout can be ineffi‐
cient, because a disk seek may be required for every page that is read. Many Btree implementations therefore try to lay out the tree so that leaf pages appear in
sequential order on disk. However, it’s difficult to maintain that order as the tree
grows. By contrast, since LSM-trees rewrite large segments of the storage in one
go during merging, it’s easier for them to keep sequential keys close to each other
on disk

        - 4.For the above reason, each leaf page may
have references to its sibling pages to the left and right, which allows scanning
keys in order without jumping back to parent pages

        - 5.B-tree variants such as fractal trees [22] borrow some log-structured ideas to
reduce disk seeks (and they have nothing to do with fractals).


- ### Comparing B-Trees and LSM-Tree

    - benchmarks are often inconclusive and sensitive to details of the workload.
You need to test systems with your particular workload in order to make a valid com‐
parison. In this section we will briefly discuss a few things that are worth considering
when measuring the performance of a storage engine.

    - A B-tree index must write every piece of data at least twice: once to the write-ahead
log, and once to the tree page itself (and perhaps again as pages are split). There is
also overhead from having to write an entire page at a time, even if only a few bytes in
that page changed. One write to the database resulting in multiple
writes to the disk over the course of the database’s lifetime—is known as write ampli‐
fication. (Although Log-structured indexes also rewrite data multiple times due to repeated compaction
and merging of SSTables)

    - Advantage of LSM Tree

        -  LSM-trees are typically able to sustain higher write throughput than Btrees, partly because they sometimes have lower write amplification n (although this
depends on the storage engine configuration and workload) and partly because they
sequentially write compact SSTable files rather than having to overwrite several pages
in the tree. This difference is particularly important on magnetic hard drives,
where sequential writes are much faster than random writes. (But the firmware internally uses a log-structured algorithm to turn ran‐
dom writes into sequential writes on the underlying storage chips, so the impact of
the storage engine’s write pattern is less pronounced)

        - LSM-trees can be compressed better. B-tree storage engines leave some disk space unused due to fragmenta‐
tion: when a page is split or when a row cannot fit into an existing page, some space
in a page remains unused. Since LSM-trees are not page-oriented and periodically
rewrite SSTables to remove fragmentation, they have lower storage overheads, espe‐
cially when using leveled compaction

        - LSM-trees can make better use of available I/O
bandwidth. Lower write
amplification and reduced fragmentation are still advantageous on SSD.

    - Downsides of LSM-tree:

        - The compaction process can sometimes
interfere with the performance of ongoing reads and writes.  The impact on
throughput and average response time is usually small, but at higher percentile the response time of queries to log-structured
storage engines can sometimes be quite high, and B-trees can be more predictable.

        - Another issue with compaction arises at high write throughput: the disk’s finite write
bandwidth needs to be shared between the initial write (logging and flushing a
memtable to disk) and the compaction threads running in the background. But the bigger the database gets, the more disk bandwidth is required for compaction. If write throughput is high and compaction is not configured carefully, it can happen
that compaction cannot keep up with the rate of incoming writes. In this case, the
number of unmerged segments on disk keeps growing until you run out of disk
space, and reads also slow down because they need to check more segment files. Typ‐
ically, SSTable-based storage engines do not throttle the rate of incoming writes, even
if compaction cannot keep up, so we need explicit monitoring to detect this situa‐
tion

    - ??? An advantage of B-trees is that each key exists in exactly one place in the index,
whereas a log-structured storage engine may have multiple copies of the same key in
different segments. This aspect makes B-trees attractive in databases that want to
offer strong transactional semantics: in many relational databases, transaction isolation is implemented using locks on ranges of keys, and in a B-tree index, those locks
can be directly attached to the tree

    - B-trees are very ingrained in the architecture of databases and provide consistently
good performance for many workloads, so it’s unlikely that they will go away anytime
soon. Although In new datastores, log-structured indexes are becoming increasingly popular.
There is no quick and easy rule for determining which type of storage engine is better
for your use case, so it is worth testing empirically. 

- ### Other Indexing Structures

    - The above key-value index have been discussed which like a primary key index. A secondary index also  can easily be constructed from a key-value index. The key in an index is the thing that queries search for, but the value can be one of
two things: it could be the actual row (document, vertex) in question, or it could be a
reference to the row stored elsewhere 

    - It can be desirable to store the indexed row directly
within an index. This is known as a **clustered index** (in MySQL’s
InnoDB storage engine, the primary key of a table is always a clustered index, and
secondary indexes refer to the primary key)

    - A compromise between a clustered index (storing all row data within the index) and
a nonclustered index (storing only references to the data within the index) is known
as a **covering index** or index with included columns, which stores some of a table’s col‐
umns within the index (in which case, the index is said to cover the query)

- ### Multi-column indexes

    - The most common type of multi-column index is called a concatenated index, which
simply combines several fields into one key by appending one column to another (the
index definition specifies in which order the fields are concatenated)

    - Multi-dimensional indexes are a more general way of querying several columns at
once, which is particularly important for geospatial data.(eg:  looking at the restaurants on a map, the website needs to
search for all the restaurants within the rectangular map area /  on an ecommerce website you could use a three-dimensional
index on the dimensions (red, green, blue) ) 

        - PostGIS implements geospatial indexes as R-
trees using PostgreSQL’s Generalized Search 
Tree indexing facility

        -  A 2Dindex could 
narrow down by timestamp and temperature 
simultaneously. This technique is used by 
HyperDex. 

        - But a standard B-tree or LSM-tree index is not 
able to answer that kind of query efficiently: it 
can give you either all the restaurants in a range 
of latitudes (but at any longitude), or all the 
restaurants in a range of longitudes (but 
anywhere between the North and South poles), 
but not both simultaneously.

- ### Full-text search and fuzzy indexes

    - Lucene is able to search text for 
wordswithin a certain edit distance (an edit 
distance of 1 means that one letter has been added/removed/replaced 

    - Other fuzzy search techniques go in the 
direction of document classification and
machine learning. See an information retrieval 
textbook for more detail

- ### Keeping everything in memory

    - Some in-memory 
databases aim for durability， by writing a log of changes to 
disk, by writing periodic snapshots to disk, or 
by replicating the in-memory state to other 
machines. Despite writing to 
disk, it’s still an in-memory database, **because 
the disk is merely used as anappend-only log 
for durability, and reads are served entirely 
from memory.**

    - Counterintuitively, the performance advantage of in-memory databases is not due to
the fact that they don’t need to read from disk. Even disk-based storage engine may
never need to read from disk if you have enough memory, because the operating sys‐
tem caches recently used disk blocks in memory anyway. Rather, they can be faster
because they can avoid the overheads of encoding in-memory data structures in a
form that can be written to disk.

    - Besides performance, another interesting area for in-memory databases is providing
data models that are difficult to implement with disk-based indexes. For example,
Redis offers a database-like interface to various data structures such as priority
queues and sets. Because it keeps all data in memory, its implementation is compara‐
tively simple.

    - LRU strategy is similar to what operating systems do with virtual memory and swap
files, but the in-memory database can manage memory more efficiently than the OS, as it can
work at the granularity of individual records rather than entire memory pages. It still requires indexes to fit entirely in memory, though (like the Bitcask
example at the beginning of the chapter). Further changes to storage engine design will probably be needed if non-volatile
memory (NVM???) technologies become more widely adopted



- ### Transaction Processing or Analytics?

    - At first, the same databases were used for both transaction processing(OLTP) and analytic
queries(OLAP) . Nevertheless, in the late 1980s and early
1990s, there was a trend for companies to stop using their OLTP systems for analytics
purposes, and to run the analytics on a separate database instead. This separate data‐
base was called a **data warehouse**.

    - Snowflake schemas are more normalized than star schemas,
but star schemas(Figure 3-9) are often preferred because they are simpler for analysts to work
with. In a typical data warehouse, tables are often very wide: fact tables often have over 100
columns, sometimes several hundred [51]. Dimension tables can also be very wide, as
they include all the metadata that may be relevant for analysis

    - data warehouse 顾虑：
data warehouse queries that need to scan over millions of rows, a big bottleneck is the bandwidth for getting data from disk into memory. However, that is not the only bottleneck. Developers of analytical databases also worry about efficiently using the bandwidth from main memory into the CPU cache, avoiding branch mispredic‐ tions and bubbles in the CPU instruction processing pipeline, and making use of single-instruction-multi-data (SIMD) instructions in modern CPUs


- ### Column-Oriented Storage

    - eg: **storage of facts table** (the center of the star schemas)


- ### Sort Order in Column Storage

    - Note that it wouldn’t make sense to sort each column independently, because then we would no longer know which items in the columns belong to the same row. We can only reconstruct the whole row

    - Another advantage of sorted order is that it can help with compression of columns. If the primary sort column does not have many distinct values, then after sorting, it will have long sequences where the same value is repeated many times in a row. A simple run-length encoding, like we used for the bitmaps in Figure 3-11
That compression effect is strongest on the first sort key. The second and third sort keys will be more jumbled up， Columns further down the sorting priority appear in essentially random order, so they probably won’t compress as well

    - Different queries benefit from different sort orders, so why not store the same data sorted in several different ways. You might as well store that redundant data sorted in different ways so that when you’re processing a query, you can use the version that best fits the query pattern.Having multiple sort orders in a column-oriented store is a bit similar to having mul‐ tiple secondary indexes in a row-oriented store. Row-oriented store keeps every row in one place, secondary indexes just contain pointers to the matching rows. In a column store, there normally aren’t any pointers to data elsewhere, only columns containing values.

- ### Writing to Column-Oriented Storage

    - An update-in-place approach, like B-trees use, is not possible with compressed col‐ umns. If you wanted to insert a row in the middle of a sorted table, you would most likely have to rewrite all the column files. As rows are identified by their position within a column, the insertion has to update all columns consistently. So a good solution is LSM tree, All writes first go to an in-memory store. It doesn’t matter whether the in-memory store is row-oriented or column-oriented. When enough writes have accumulated, they are merged with the column files on disk and written to new files in bulk

- ### Aggregation: Data Cubes and Materialized Views

    - Another aspect of data warehouses that is worth mentioning briefly is materialized aggregates. As discussed earlier, data warehouse queries often involve an aggregate function, such as COUNT, SUM, AVG, MIN, or MAX in SQL. Why not cache some of the counts or sums that queries use most often?

    - The difference is that a materialized view is an actual copy of the query results, written to disk, whereas a virtual view in relational data model is just a shortcut for writing queries.

    - When the underlying data changes, a materialized view needs to be updated, because it is a denormalized copy of the data. The database can do that automatically(Stream Processing ???), but such updates make writes more expensive, which is why materialized views are not often used in OLTP databases.

    - A common special case of a materialized view is known as a data cube or OLAP cube.  It is a grid of aggregates grouped by different dimensions.(Figure 3-12)
The advantage of a materialized data cube is that certain queries become very fast because they have effectively been precomputed

    - The disadvantage is that a data cube doesn’t have the same flexibility as querying the raw data. For example, there is no way of calculating which proportion of sales comes
from items that cost more than $100 Most data warehouses therefore try to keep as much raw data as possible, and use aggregates such as data cubes only as a performance boost for certain queries.


- ### Summary
    - On the OLTP side, we saw storage engines from two main schools of thought

    - The log-structured school, which only permits appending to files and deleting obsolete files, but never updates a file that has been written. Bitcask, SSTables, LSM-trees, LevelDB, Cassandra, HBase, Lucene, and others belong to this group

    - The update-in-place school, which treats the disk as a set of fixed-size pages that can be overwritten. B-trees are the biggest example of this philosophy, being used in all major relational databases and also many nonrelational ones

    - This data warehouse. background illustrated why analytic workloads are so different from OLTP: when your queries require sequentially scan‐ ning across a large number of rows, indexes are much less relevant

    - 索引场景应对读多，disk seek time sensitive, 所以需要找到特定的那个page  方便update inplaced
但对于需要大量扫描的OLAP  disk bandwidth sensitive, 所以不要把整行读进来再根据字段判断筛选，column oriented db更加有效地利用了每个colunm 的distinct value的位图及位图之间的位运算实现了高效的筛选；同时，排序后的位图能利用长度编码实现高效压缩存储。高效利用磁盘IO

    - It has hopefully equipped you with enough vocabulary and ideas that you can make sense of the documentation for the database of your choice Nice








