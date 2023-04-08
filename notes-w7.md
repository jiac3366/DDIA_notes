
- ### Leaderless Replication

    - ### Writing to the Database When a Node Is Down

        - What should we do if we read in leaderless database or with a node failure ?
        - In this case, read requests are also sent to several nodes in parallel. The client may get different responses from different nodes,Version numbers are used to determine
          which value is newer (**Read repair**). And client writes the newer value back to that stale replica.

        - In addition, some datastores have a background **anti-entropy** process that constantly looks for
          differences in the data between replicas and copies any missing data from one
          replica to another. If without an anti-entropy process, values that
          are rarely read may be missing from some replicas and thus have reduced durability

    - ### Quorums for reading and writing

        -  Reads and writes that obey these r and w values are called quorum reads
          and writes (In our
          example, n = 3, w = 2, r = 2.) If w + r > n, at least one of the r replicas you read from must have seen the
          most recent successful write. 读和写的节点是有交集的 A
          common choice is to make n an odd number (typically 3 or 5) and to set w = r =
          (n + 1) / 2 (rounded up)

        -  For
          example, a workload with few writes and many reads may benefit from setting w = n
          and r = 1. This makes reads faster, but has the disadvantage that just one failed node
          causes all database writes to fail. With n = 3, w = 2, r = 2 we can tolerate one unavailable node; With n = 5, w = 3, r = 3 we can tolerate two unavailable nodes.  If fewer than the required w or r nodes are available, writes or reads return an error.

        -  It only matters that the sets of nodes used by the read and
          write operations overlap in at least one node, other quorum assignments are possi‐
          ble.  Only after the number of reachable replicas
          falls below w or r does the database become unavailable for writing or reading,
          respectively.

        -  However, even with w + r > n, there are likely to be edge cases where stale values are
          returned (**Limitations of Quorum Consistency**):

            - sloppy quorum?

            - need to detecting concurrent writes

            - If a write happens concurrently with a read, read  returns the
            old or the new value is undetermined

            - write succeeded on some replicas but failed on others, it is not rolled back on the replicas where it succeeded. subsequent read  returns the
            old or the new value is undetermined

            - if node carry with new value fail and then it  restored from a replica carry‐
            ing an old value, will breaking the w quorum condition.

            - n “Linearizability and quorums”??





        - In practice it is not so simple to guarantee a read returns the latest written
value in "Quorums", and also usually do not get the guarantees from (reading your writes, monotonic reads, or consistent prefix)

    - ### Monitoring staleness
    
        -  This is possible because writes are
applied to the leader and to followers in the same order, and each node has a position
in the replication log (the number of writes it has applied locally). By subtracting a
follower’s current position from the leader’s current position, you can measure the
amount of replication lag ? However, in systems with leaderless replication, there is no fixed order in which
writes are applied, which makes monitoring more difficult

        - It would be good to include staleness measurements in the standard set of metrics for
databases because there has been some research on measuring replica staleness in databases with leaderless replication and predicting the expected percentage of stale reads depending on
the parameters n, w, and r

- ### Sloppy Quorums and Hinted Handoff

    - In a large cluster (with significantly more than n nodes because of the partition?), we accept writes anyway, and write them to some nodes that are
      reachable but aren’t among the n nodes on which the value usually lives. This case calls **sloppy quorum** 有些分片集群节点总数多于指定的n个复制集数，所以写的值可以暂时在n之外的节点待一下，提高了高可用。 However, this means that even
      when w + r > n, you cannot be sure to read the latest value for a key 但影响了读最新值的正确性，因为最新的值待在n以外的节点中

    - Once the network interruption is fixed, any writes that one node temporarily
      accepted on behalf of another node are sent to the appropriate “home” nodes. This is called **hinted handoff**. 最新的值从n以外的节点回到n个节点（暗示的交接）

    - ### Multi-datacenter operation ??? page 184

- ### Detecting Concurrent Writes

    let’s explore the issue in a bit
    more detail.

    - Last write wins (discarding concurrent writes) 

        - only one of the writes
          will survive and the others will be silently discarded. Moreover, LWW may even drop
          writes that are not concurrent, as we shall discuss in “Timestamps for ordering
          events” on page 291. If losing data is not acceptable, LWW is a poor choice for conflict resolution.

        - LWW achieves the goal of eventual convergence, only one of the writes
          will survive and the others will be silently discarded. The only safe way of using a database with LWW is to ensure that a key is only written once and thereafter treated as immutable, thus avoiding any concurrent updates
          to the same key. For example, a recommended way of using Cassandra is to use a
          UUID as the key, thus giving each write operation a unique key [53]. ???

    - The “happens-before” relationship and concurrency

        **Whether one operation happens before another
        operation is the key to defining what concurrency means.** What we
        need is an algorithm to tell us whether two operations are concurrent or not

        - 1. In Figure 5-9, the two writes are not concurrent,  B is causally dependent on A because B’s operation builds upon A’s operation, so B’s operation must have
            happened later.

        - 2. the two writes in Figure 5-12 are concurrent: each cli‐
            ent starts the operation, it does not know that another client is also performing
            an operation on the same key

        - It is actually quite difficult to
          tell whether two things happened at exactly the same time—an issue we will discuss
          in more detail in Chapter 8. For defining concurrency, exact time doesn’t matter: we simply call two operations
          concurrent if they are both unaware of each other, regardless of the physical time at
          which they occurred.

    - Capturing the happens-before relationship

        Let’s look at an algorithm that determines whether two operations are concurrent, or
        whether one happened before another.

        - Figure 5-13 / 5-14. 核心思想是客户端做union操作，并且在本地维护版本好上一次写操作的版本号，这样，下次写操作将上一次的写操作一起发送给DB。这种算法和simple approach is to just pick one of the values based on a version
          number or timestamp (last write wins) 不同，因为后者会丢数据

        - When the server receives a write with a particular version number, it can over‐
          write all values with that version number or below. 你只有收到版本2，才能重写版本2或以下的值. When a write includes the version number from a prior read, that tells us which pre‐
          vious state the write is based on. If you make a write without including a version
          number, it is concurrent with all other writes, so it will not overwrite anything 

        - For the add case, we can take the union operation for this add response but could not apply the union operation on the delete response because removed item will reappear. To figure out this issue, the system must leave a marker with an appropriate version number to indicate that
          the item has been removed when merging siblings. There are some
          efforts to design data structures that can perform this merging automaticallyFor example, Riak’s datatype
          support uses a family of data structures called CRDTs [38, 39, 55] that can automati‐
          cally merge siblings in sensible ways, including preserving deletions.


    - ### Version vectors
    
        The example in Figure 5-13 used only a single replica. How does the algorithm
change when there are multiple replicas, but no leader?

        - Figure 5-13 uses a single version number to capture dependencies between operations,  that is not sufficient when there are multiple replicas accepting writes concurrently. Instead, we need to use a version number per replica as well as per key.
Each replica increments its own version number when processing a write, and also keeps track of the version numbers it has seen from each of the other replicas. 客户端需要对每个复制集和每个key记录下其版本号，就像购物车的案例一样。只不过购物车案例只有一个复制集合。This collection of version numbers from all the replicas is called a **version vector**.  Riak 2.0在使用一些相似的idea算法。The version vector allows the database to distinguish between
overwrites and concurrent writes. In brief, when comparing the
state of replicas, version vectors are the right data structure to use


- ### Summary 

    - Each approach has advantages and disadvantages. Single-leader replication is popular
      because it is fairly easy to understand and there is no conflict resolution to worry
      about. Multi-leader and leaderless replication can be more robust in the presence of faulty nodes, network interruptions, and latency spikes—at the cost of being harder
      to reason about and providing only very weak consistency guarantees.


## Chapter 6 Partitioning

Some systems are designed for transactional work‐
loads, and others for analytics, this difference affects how the system is tuned, but the fundamentals of partitioning apply to both kinds of workloads.


Figure 6-1. Combining replication and partitioning: each node acts as leader for some
partitions and follower for other partitions. 

If the partitioning is unfair, so that some partitions have more data or queries than others, we call it skewed. A partition with disproportionately high load is called a hot spot.

- ### Partitioning of Key-Value Data
    - ### Partitioning by Key Range
        - The partition boundaries might be chosen manually by an administrator, or the data‐
          base can choose them automatically 因为分区的分界需要和数据相适应，按照例子Figure 6-2， 即便是每2个字母一个分区也会导致其中分区倾斜

<<<<<<< HEAD
        - Within each partition, we can keep keys in sorted order. This has the advantage that range scans are easy, and you can treat the key as a concatenated index in order to fetch several related records in one
          query. 排好的数据使得range scan更加高效, 因为容易得知数据的边界；如果数据按照多个key排好序，那么按照多个key的range scan也会高效，多重索引原理也是如此
=======
        - Within each partition, we can **keep keys in sorted order**( a partition owns all the keys
from some minimum up to some maximum). This has the advantage that range scans are easy, and you can treat the key as a concatenated index in order to fetch several related records in one query. 按照排好的key分区使得range scan更加高效, 因为容易得知分区的数据边界；如果按照排好序的多个key分区，那么按照多个key的range scan也会高效。多重索引的原理也是如此。
>>>>>>> 574f2b704b4bea411ac8372fdf92008e8ca4a057

        - 如何选择好分区的key: one partition per day will lead to one partition can be overloaded with writes on one day
          while others sit idle。 单一高层次维度的key（时间某一天）会造成热点节点，所以可以在key中加入一些其他分散维度的key（某一个传感器）。So we could prefix each
          timestamp with the sensor name so that the partitioning is first by sensor name and
          then by time. Assuming we have many sensors active at the same time, the write
          load will end up more evenly spread across the partitions. Now, when you want to
          fetch the values of multiple sensors within a time range, you need to perform a sepa‐
          rate range query for each sensor name

    - Advantage of Partitioning by Key Range is Fastly range scan; Disvantage is certain access patterns can
      lead to **hot spots**

    - ### Partitioning by Hash of Key

        - We should have  A good hash function which such as: Whenever you give it a new string, it
<<<<<<< HEAD
          returns a seemingly random number between 0 and 232 − 1, Once you have a suitable hash function for keys, you can assign each partition a
          range of hashes (rather than a range of keys) every key whose hash falls within a
          partition’s range will be stored in that partition. The partition
          boundaries can be evenly spaced, or they can be chosen pseudorandomly (in which
          case the technique is sometimes known as consistent hashing).
=======
returns a seemingly random number between 0 and 232 − 1, Once you have a suitable hash function for keys, you can assign each partition **a
range of hashes** (rather than a range of keys) every key whose hash falls within a
partition’s range will be stored in that partition. The partition
boundaries can be evenly spaced, or they can be chosen pseudorandomly (in which
case the technique is sometimes known as consistent hashing). 注意，每个数据的key落在某个区间，就去到某个分区；可以减小节点增加或删除时rebalancing的规模？分区数一般比节点数要大，而且尽量大点，
>>>>>>> 574f2b704b4bea411ac8372fdf92008e8ca4a057

        - Consistent Hashing: It uses randomly chosen partition boundaries to avoid the need for central control or
          distributed consensus. Note that consistent here has nothing to do with replica consis‐
          tency (see Chapter 5) or ACID consistency (see Chapter 7), but rather describes a
          particular approach to rebalancing. 一种将数据均匀分布的算法

        - The pros and crons of Partitioning by Hash of Key is opposite of Partitioning by Hash of Key.  
<<<<<<< HEAD

            - In MongoDB, if you have enabled hash-based sharding mode, any range query
              has to be sent to all partitions

            - Range queries on the primary key are not sup‐
              ported by Riak, Couchbase, or Voldemort.

            - Cassandra achieves a compromise between the two partitioning strategies . A table in Cassandra can be declared with a compound primary key consisting of
              several columns. The first part of that key is hashed to determine the partition,
              but the other columns are used as a concatenated index for sorting the data in Cassandra’s SSTables. A query therefore cannot search for a range of values within the first column of a compound key, but if it specifies a fixed value for the first column, it
              can perform an efficient range scan over the other columns of the key. 多列作为主键的情况下，如果使用第一列进行分区，并且在分区内的该列都相同，那么就可以对后面的列进行范围查询.
=======
        
            - In MongoDB, if you have enabled hash-based sharding mode, any range query as to be sent to all partitions

            - Range queries on the primary key are not supported by Riak, Couchbase, or Voldemort.

            - Cassandra achieves a compromise between the two partitioning strategies （Hybrid approaches）. A table in Cassandra can be declared with a compound primary key consisting of
several columns. The first part of that key is hashed to determine the partition,
but the other columns are used as a concatenated index for sorting the data in Cassandra’s SSTables. A query therefore cannot search for a range of values within the first column of a compound key, but if it specifies a fixed value for the first column, it
can perform an efficient range scan over the other columns of the key. Cassandra是SSTables为底层结构，其他列用做SSTables的索引了，它在固定第一列（分区key）的基础上, 对后面的列可以进行高效的范围查询.
>>>>>>> 574f2b704b4bea411ac8372fdf92008e8ca4a057

        - A celebrity user with millions of followers may cause a storm of
          activity when they do something. Sulotion: For
          example, if one key is known to be very hot, a simple technique is to add a random
          number to the beginning or end of the key, Just a two-digit decimal random number
          **would split the writes to the key evenly across 100 different keys, allowing those keys
          to be distributed to different partitions.**  为了防止明星出轨之类的事件干掉数据库，可以在生成分区键时在末尾添加2位随机数，这时候就把写操作的key分成了多达100个不同的key（不是100个节点，有可能是少于100个）从而这些key会被分布到不同的分区？However, having split the writes across different keys, any reads now have to do addi‐
          tional work, as they have to read the data from all 100 keys and combine it。 Thus, you also need some
          way of keeping track of which keys are being split.  读的时候得做额外工作，把这100个key的内容读回来并合并，因此要记录下key被拆分成了哪些随机的key，这些仅仅对于热点key有意义。This approach to querying a partitioned database is sometimes known as **scatter/
          gather**.   Even if
          you query the partitions in parallel, scatter/gather is prone to tail latency amplifica‐
          tion (see “Percentiles in Practice” on page 16). Need to think through the trade-offs for
          your own application.

- ### Partitioning and Secondary Indexes

    There are two main approaches to partitioning a database with secondary indexes:
    document-based partitioning and term-based partitioning.

    - ### Partitioning Secondary Indexes by Document

<<<<<<< HEAD
        - As shown in Figure 6-4, In this indexing approach, each partition is completely separate: each partition main‐
          tains its own secondary indexes, covering only the documents in that partition. It
          doesn’t care what data is stored in other partitions, As mentioned above, read will be slow cause need to combine the result.  Nevertheless, it is widely used: Mon‐
          goDB, Riak [15], Cassandra [16], Elasticsearch [17], SolrCloud [18], and VoltDB [19]
          all use document-partitioned secondary indexes
=======
        - As shown in Figure 6-4, In this indexing approach, each partition is completely separate: each partition maintains its own secondary indexes, covering only the documents in that partition. It
doesn’t care what data is stored in other partitions, As mentioned above, read will be slow cause need to combine the result.  Nevertheless, it is widely used: Mon‐
goDB, Riak [15], Cassandra [16], Elasticsearch [17], SolrCloud [18], and VoltDB [19]
all use document-partitioned secondary indexes
>>>>>>> 574f2b704b4bea411ac8372fdf92008e8ca4a057

        - Most database vendors recommend
          that you structure your partitioning scheme so that secondary index queries can be
          served from a single partition, but that is not always possible, especially when you’re
          using multiple secondary indexes in a single query (such as filtering cars by color and
          by make at the same time).  大多数数据库vendor说调整分区schemas会有助于query只到1个节点上，事实上不太可能，特别是同时查询多个field的时候。

    - ### Partitioning Secondary Indexes by Term (by itself)

        - As shown in Figure 6-4, A global index must also be partitioned, but it can be partitioned
differently from the primary key index.  We call this kind of ***index term-partitioned*** because the term we’re looking for determines the partition of the index

        - As before, we can partition the index by the term itself, or using a hash of the term.

        - The advantage of a global (term-partitioned) index over a document-partitioned index is that it can make reads more efficient: rather than doing scatter/gather over all partitions, a client only needs to make a request to the partition containing the term that it wants. However, the downside of a global index is that writes are slower and more complicated, because a write to a single document may now affect multiple partitions of the index (every term in the document might be on a different partition on a different node). In a term partitioned index, that would require a distributed transaction across all partitions affected by a write, which is not supported in all databases 总结两种分区存储索引的方式：按照文档（数据）分区，读慢写快；按照索引项分区，读快写慢.  Amazon DynamoDB, Riakk’s search feature and the Oracle data warehouse, use global term-partitioned indexes

        - will return to the topic of implementing term-partitioned secondary indexes
in Chapter 12.


- ### Rebalancing Partitions

    - rebalancing is usually expected to meet
some minimum requirements:

        - After rebalancing, the load (data storage, read and write requests) should be
shared fairly between the nodes in the cluster.
        - While rebalancing is happening, the database should continue accepting reads
and writes.
        - No more data than necessary should be moved between nodes,  to make rebalanc‐
ing fast and to minimize the network and disk I/O load.

    - ### Strategies for Rebalancing

        - The problem with the mod N (number of the node) approach lead to  frequent data moves if we add node, which make rebalancing excessively
expensive

        - So a fairly simple solution (Figure  6-6): We can set a database running on a cluster of 10 nodes may be split into 1,000 partitions. ( create many more partitions than there
are nodes) if add node, just steal some data from every exist nodes to make each node rebalanced. The number of partitions does not
changee, nor does the assignment of keys to partitions. This change of assignment is not immediate—
it takes some time to transfer a large amount of data over the network—so the old
assignment of partitions is used for any reads and writes that happen while the transfer is in progress.

        - ### Fixed number of partitions
            - It's simpler for the number of partitions is usually fixed when the database is
    first set up and not changed afterward. you need to choose it high enough to accommo‐
    date future growth. However, each partition also has management overhead, so it’s
    counterproductive to choose too high a number. I think we should keep the size of each partition is “just right”. However, it's hard to archive if the number of partitions is fixed but the dataset size
    varies. 

            -  ps: you can even account for mismatched hardware in your cluster: by
    assigning more partitions to nodes that are more powerful, you can force those nodes
    to take a greater share of the load. 

        - ### Dynamic partitioning

            - An advantage of dynamic partitioning is that the number of partitions adapts to the
total data volume. If there is only a small amount of data, a small number of parti‐
tions is sufficient, so overheads are small; if there is a huge amount of data, the size of
each individual partition is limited to a configurable maximum (Just like B-tree, on
HBase) When a partition grows to exceed a configured size (on
HBase, the default is 10 GB)  it is split into two partitions. Conversely, It will merge with an adjacent par‐
tition.

            - A caveat is the dataset is small(or empty db) on the beginning, until it hits the point at which the first partition is split, all
writes have to be processed by a single node while the other nodes sit idle. So HBase and MongoDB allow an initial set of partitions to be configured
on an empty database (this is called pre-splitting). So pre-splitting requires that you already know what the key distribution is going to
look like. 
            - Dynamic partitioning is not only suitable for key range–partitioned data, but can
equally well be used with hash-partitioned data. MongoDB since version 2.4 supports
both key-range and hash partitioning, and it splits partitions dynamically in either
case.

        - ### Partitioning proportionally to nodes

            - For the above 2 cases, the number of partitions is independent of the number of nodes. A third option, used by Cassandra and Ketama, is to make the number of partitions
proportional to the number of nodes,  this approach also
keeps the size of each partition fairly stable. 分区数和节点数成正比 Cassandra 3.0 introduced an alternative rebalanc‐
ing algorithm that avoids unfair splits

            - Picking partition boundaries randomly requires that hash-based partitioning is used, Indeed, this approach corresponds most closely to the original definition
of consistent hashing ?

    - ### Operations: Automatic or Manual Rebalancing

        -  Rebalancing can
overload the network or the nodes and harm the performance of other requests while
the rebalancing is in progress, For example, say one node is overloaded and is temporarily slow to respond to
requests. The other nodes conclude that the overloaded node is dead, and automati‐
cally rebalance the cluster to move load away from it. This puts additional load on the
overloaded node, other nodes, and the network—making the situation worse and
potentially causing a cascading failure

        - For that reason, it can be a good thing to have a human in the loop for rebalancing.


- ### Request Routing

    - Figure 6-7. Three different ways of routing a request to the right node

    - Many distributed data systems rely on a separate coordination service such as Zoo‐
Keeper to keep track of this cluster metadata, as illustrated in Figure 6-8. Each node
registers itself in ZooKeeper, and ZooKeeper maintains the authoritative mapping of
partitions to nodes. Other actors, such as the routing tier or the partitioning-aware
client, can subscribe to this information in ZooKeeper. Whenever a partition changes
ownership, or a node is added or removed, ZooKeeper notifies the routing tier so that
it can keep its routing information up to date. HBase, SolrCloud, and Kafka also use ZooKeeper to track partition assignment.
MongoDB has a similar architecture, but it relies on its own config server implemen‐
tation and mongos daemons as the routing tier. 

    - ### Parallel Query Execution

        - massively parallel processing (MPP) relational database products, often
used for analytics, are much more sophisticated in the types of queries they support.
A typical data warehouse query contains several join, filtering, grouping, and aggregation operations. The MPP query optimizer breaks this complex query into a number of execution stages and partitions, many of which can be executed in parallel on
different nodes of the database cluste

    - However, Partition means operations need to write to several partitions. What happens if the
write to one partition succeeds, but another fails? We will address that question in the
following chapters.




