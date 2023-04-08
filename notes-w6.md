
## Part 2 Distributed Data


- ### Scaling to Higher Load

    - one the simplest approach is to buy a more pow‐
erful machine( sometimes called vertical scaling or scaling up) shared-memory architecture, all the components can be treated as a single
machine 

    - Another approach is the shared-disk architecture, which uses several machines with
independent CPUs and RAM, but stores data on an array of disks that is shared
between the machiness, which are connected via a fast network.( MO: Amazon EC2 and S3 file system ?) —— Network Attached Storage (NAS) or Storage Area Network (SAN). 


- ### Shared-Nothing Architectures

    - shared-nothing architectures has many advantages and  can be very powerful (sometimes called horizontal scaling or scaling out) In this approach, each machine or virtual
machine running the database software is called a node. Each node uses its CPUs, RAM, and disks independently. Any coordination between nodes is done at the software level, using a conventional network. No special hardware is required by a shared-nothing system. In this part of the book, we focus on shared-nothing architectures because they require the most caution from you, the application developer.  But first, let’s talk about distributed data

## Chapter 5 Replication

- **Why you want to replicate data?**

    - To keep data geographically close to your users (and thus reduce latency)
    - To allow the system to continue working even if some of its parts have failed (increase availability)
    - To scale out the number of machines that can serve read queries (increase read throughput)


- ### Leaders and Followers

    - leader-based replication
is not restricted to only databases: distributed message brokers such as Kafka and
RabbitMQ highly available queues also use it. Some network filesystems and
replicated block devices such as DRBD are similar.

    -  There are circumstances when followers might fall behind the leader by several minutes or more; for example, if a follower is recovering from a failure, if the system is operating near maximum capacity, or if there are network problems between the nodes

    - **Synchronous replication pros and cons**

        - guaranteed to have an up-to-date copy of the data that is consistent with the leader. If the leader suddenly fails, we can be sure that the data is still available on the follower.

        - The disadvantage is that if the synchronous follower doesn’t respond, the write cannot be processed. The leader must block all writes and wait until the synchronous replica is available again

    - **In practice, if you enable synchronous replication on a database, it usually means that one of the followers is synchronous, and the others are asynchronous. If the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous. This guarantees that you have an up-to-date copy of the data on at least two nodes: leader and one synchronous follower (semi-synchronous)**

    - **But Oftenly if some data "loss" will not influence business, leader-based replication is configured to be completely asynchronous**. Especially if there are many followers or if they are geographically distributed. Even though a write is not guaranteed to be durable( any writes that have not yet been replicated to followers are lost)

     - If add a follower, How to ensure that the new follower
has an accurate copy of the leader’s data?  

        - Due to the clients
are constantly writing to the database, and the data is always in flux, so a standard file
copy would see different parts of the database at different points in time, Simply copying data files from one node to another
might not make any sense

        1. Take a consistent snapshot of the leader’s database at some point in time,  Most databases have this fea‐
ture, as it is also required for backups. (In some cases, third-party tools are
needed, such as innobackupex for MySQL)

        2. Copy the snapshot to the new follower node

        3. The follower connects to the leader and requests all the data changes that have
happened since the snapshot was taken.This requires that the snapshot is associ‐
ated with an exact position in the leader’s replication log. That position has vari‐
ous names: for example, PostgreSQL calls it the **log sequence number**, and
MySQL calls it the **binlog coordinates**.

        4. When the follower has processed the backlog of data changes since the snapshot,
we say it has **caught up**.


- ### Handling Node Outages

    - **Follower failure: Catch-up recovery**: the follower can recover quite easily:
from its log, it knows the last transaction that was processed before the fault occurred.  the follower can connect to the leader and request all the data changes that occurred during the time when the follower was disconnected

    - **Leader failure: Failover**: Handling a failure of the leader is trickier: one of the followers needs to be promoted
to be the new leader,  clients need to be reconfigured to send their writes to the new
leader, and the other followers need to start consuming data changes from the new
leader. This process is called **failover**. 故障切换

        - Failover can happen manually (an administrator is notified that the leader has failed
and takes the necessary steps to make a new leader)

        - An automatic failover process usually consists of the following steps：

            1. Determining that the leader has failed， if a node doesn’t respond for some period of time—say, 30 seconds—it is
    assumed to be dead  心跳检测

            2. Choosing a new leader. Could be done through an election process (chosen by  majority of the remaining replicas )or be appointed by a previously elected **controller node**???  The best candidate for
    leadership is usually the replica with the most up-to-date data changes from the
    old leader. Getting all the nodes to agree on a new
    leader is a consensus problem, discussed in detail in Chapter 9.  重新选举 / 共识问题

            3. Reconfiguring the system to use the new leader. .If the old leader comes back, it might still believe that it is the leader,
    not realizing that the other replicas have forced it to step down. The system
    needs to ensure that the old leader becomes a follower and recognizes the new
    leader  

        - Failover is **fraught** with things that can go wrong:

            1. If asynchronous replication is used,** the new leader may not have received all the
writes from the old leader before it failed**.  The most common
solution is for the old leader’s unreplicated writes to simply be discarded, which
may violate clients’ durability expectations.  新leader有可能有遗漏更新 Discarding writes is especially dangerous if other storage systems outside of the
database need to be coordinated with the database contents. eg: one incident at Github, the new leader’s counter lagged behind the old leader’s, it
reused some primary keys that were previously assigned by the old leader. These
primary keys were also used in a Redis store, so the reuse of primary keys resulted in inconsistency between MySQL and Redis

            2. likely happen that two nodes both
believe that they are the leader. This situation is called **split brain**,脑裂 and it is dangerous: if both leaders accept writes, and there is no process for **resolving conflicts** (see “Multi-Leader Replication” on page 168), data is likely to be lost or
corrupted. As a safety catch, some systems have a mechanism to shut down one
node if two leaders are detected. However, if this mechanism is not carefully
designed, you can end up with both nodes being shut down

            3. If the system is already struggling with high load or network problems, an unnecessary failover is likely to
make the situation worse, not better. 使得有高负载或网络差的集群的问题更加严重。心跳检测Timeout时间要设置得十分小心？ System should define the right timeout before the leader is declared dead —— A longer timeout
means a longer time to recovery in the case where the leader fails. However, if the
timeout is too short, there could be unnecessary failovers

        - There are no easy solutions to above problems. For this reason, some operations
teams prefer to perform failovers **manually**, even if the software supports automatic
failover


- ### Implementation of Replication Logs

     Several different replica‐
tion methods are used in practice: 

    - **Statement-based replication**
    
        - For a relational database, this means
that every INSERT, UPDATE, or DELETE statement is forwarded to followers, and each follower parses and executes that SQL statement as if it had been received from a
client. Some break down are following:

            - such as NOW() to get the
    current date and time or RAND() to get a random number, is likely to generate a
    different value on each replica.

            - If statements use an autoincrementing column, or if they depend on the existing
    data in the database, they must be
    executed in exactly the same order on each replica. This can be limiting when there are multiple concurrently executing
    transactions.???

            - Statements that have side effects (e.g., triggers, stored procedures, user-defined
    functions) may result in different side effects occurring on each replica ??

        - Statement-based replication was used in MySQL before version 5.1. It is still some‐
times used today, as it is quite compact, but by default MySQL now switches to **row based replication**

    - **Write-ahead log (WAL) shipping**

        -  the log is an append-only sequence of bytes containing all writes to the
database. We can use the exact same log to build a replica on another node. Main disadvantage is that the log describes the data on a very low level: a WAL con‐
tains details of which bytes were changed in which disk blocks. This makes replica‐
tion closely coupled to the storage engine. If the database changes its storage format
from one version to another, it is typically not possible to run different versions of
the database software on the leader and the followers.

        -  If the replication protocol allows the follower to use a newer software version
than the leader, you can perform a zero-downtime upgrade of the database software
by first upgrading the followers and then performing a failover to make one of the
upgraded nodes the new leader. If the replication protocol does not allow this version
mismatch, as is often the case with WAL shipping, such upgrades require downtime.


    - **Logical (row-based) log replication**

        - use different log formats for replication and for the storage
engine, which allows the replication log to be decoupled from the storage engine
internals. This kind of replication log is called a logical log, to distinguish it from the
storage engine’s (physical) data representation. It can more easily
be kept backward compatible, allowing the leader and the follower to run different
versions of the database software, or even different storage engines.

        -  relational database is usually a sequence of records describing
writes to database tables at the granularity of a row (inserted row / deleted row / updated row)

    - **Trigger-based replication**

        - Trigger-based replication typically has greater overheads than other replication
methods, and is more prone to bugs and limitations than the database’s built-in repli‐
cation. However, it can nevertheless be useful due to its flexibility.


- ###  Solutions for Replication Lag 
    Three examples of problems that are likely to occur when there is replication lag and outline some **approaches** to solving them.

    - **Reading Your Own Writes**
        eg: the user views the data shortly after making a write, the new data may not yet have reached the replica, In this situation, we need read-after-write consistency, also known as **read-your-writes consistency**.
        - If sth only editable by user (profile of the user), always read it from leader. If most things in the application are potentially editable by the user, above approach won’t be effective. For example, you could track the time of the last update and, for one minute after the last update, make all reads from the leader. 在短时间内，全部读主节点。
        - The client can remember the timestamp of its most recent write. The system can ensure the replica serving any reads, whose update of the user has been reflected on it at that timestamp. The timestamp could be a logical timestamp (something that indicates ordering of writes, such as the log sequence number) or the actual system clock  （Not quite clear）
        - If your replicas are distributed across multiple datacenters,  Any request that needs to be served by the leader must be routed to the datacenter that contains the leader

        For the cross-device read-after-write consistency:
        - The timestamp of the user’s last update need to be centralized
        - If your approach requires reading from the leader, you may first need to route requests from all of a user’s devices to the same datacenter.

    - **Monotonic Reads**
        - eg: The first query returns a comment that was recently added by user 1234, but the second query doesn’t return anything because the lagging follower has not yet picked up that write.
    target: will not read older data after having previously read newer data
    One way of achieving monotonic reads is to **make sure that each user always makes their reads from the same replica. For example, the replica can be chosen based on a hash of the user ID,** rather than randomly. However, if that replica fails, the user’s queries will need to be rerouted to another replica.

    - **Consistent Prefix Reads**

        - eg: If some partitions are replicated slower than others, an observer may see the answer before they see the question. But different partitions operate independently, so there is no global ordering of writes: when a user reads from the database, they may see some parts of the database in an older state and some in a newer state

        - **One solution is to make sure that any writes that are causally related to each other are written to the same partition** —— but in some applications that cannot be done efficiently. There are also algorithms that explicitly keep track of causal dependencies, a topic that we will return to in “The “happens-before” relationship and concurrency” on page 186. 有点像消息队列。同一业务顺序的消息，需要路由到同一个Topic，所以读的人可以保证读到正确顺序的内容。

    - As discussion above, by performing certain kinds of reads on the leader. **However, dealing with these issues in application code is complex and easy to get wrong**. it would be better if application developers didn’t have to worry about subtle replication. **This is why transactions exist.**  Single-node transactions have existed for a long time. However, in the move to distributed (replicated and partitioned) databases, many systems have abandoned them, asserting that eventual consistency is inevitable in a scalable system. There is some truth in that statement, but it is overly simplistic, and we will develop a more nuanced view over the course of the rest of this book.


- ### Multi-Leader Replication， Use Cases for Multi-Leader Replication

    - Multi-datacenter operation

        - With a normal leader-based replication setup, the leader has to be in
one of the datacenters, and all writes must go through that datacenter. But In a multi-leader configuration, you can have a leader in each datacenter. Within each datacenter, regular leader–
follower replication is used; between datacenters, each datacenter’s leader replicates
its changes to the leaders in other datacenters (If the database is partitioned (see Chapter 6), each partition has one leader. Different partitions may have
their leaders on different nodes, but each partition must nevertheless have one leader node.)

        - Advantage: 
            - **Performance**, every write can be processed in the local datacenter
and is replicated asynchronously to the other datacenters. network delay maybe better

            - **Tolerance of datacenter outages**, In a multi-leader configuration, each datacenter can continue operating independently of the others (No need to promote a follower as leader)

            - **Tolerance of network problems**
            A single-leader configu‐
ration is very sensitive to problems in this inter-datacenter link, because writes
are made synchronously over this link.单主模式一旦主节点所在数据中心内网络出问题，就会挂。 A multi-leader configuration with asynchronous replication can usually tolerate network problems better:a temporary
network interruption does not prevent writes being processed. 双主模式，两个数据中心内并且数据中心之间网络同时出问题，才挂.

        - Some databases support multi-leader configurations by default, but it is also often
implemented with external tools, such as Tungsten Replicator for MySQL [26], BDR
for PostgreSQL, and GoldenGate for Oracle


    - Clients with offline operation

        - consider the calendar apps on your mobile phone, your laptop, and
other devices. You need to be able to see your meetings (make read requests) and
enter new meetings (make write requests) at any time, regardless of whether your
device currently has an internet connection. 

        - Image that each device is a “datacenter,”
and the network connection between them is extremely unreliable, 
every device has a local database that acts as a leader (it accepts write
requests), and there is an asynchronous multi-leader replication process (sync)
between the replicas on your devices. If you make any changes while you are
offline, they need to be synced with a server and your other devices when the device
is next online.

    - Collaborative editing

        - it has a lot in common with the previously mentioned offline editing use case. 每个用户就像离线的数据库，当需要同步时，则要处理写冲突。 When
one user edits a document, the changes are instantly applied to their local replica (the
state of the document in their web browser or client application) and asynchronously
replicated to the server and any other users who are editing the same document.

         - one method is lock on the document before a user can edit it, but this collaboration model is equivalent to single-leader
replication with transactions on the leader. another is avoid locking. This approach allows multiple users
to edit simultaneously, but it also brings all the challenges of multi-leader replication,
including requiring conflict resolution, which is the biggest problem with multi-leader replication **(Handling Write Conflicts)**

- ### Handling Write Conflicts

    - eg:  User 1 changes the title of the page from A to B, and user 2
changes the title from A to C at the same time, when the changes are asynchronously replica‐
ted, a conflict is detected

    - In principle, you could make the conflict detection synchronous —— wait for the
write to be replicated to all replicas before telling the user that the write was success‐
ful. However, by doing so, will lose the main advantage of multi-leader repli‐
cation: **allowing each replica to accept writes independently， you might as well just use single-leader replication**  (所以那还不如用单主模式)

    - dealing with conflicts is to avoid them: if the application can
ensure that **all writes for a particular record go through the same leader, then conflicts cannot occur**, we can ensure
that requests from a particular user are always routed to the same datacenter and use
the leader in that datacenter for reading and writing. But sometimes you might want to change the designated leader for a record—
perhaps because one datacenter has failed and you need to reroute traffic to another
datacenter, or perhaps because a user has moved to a different location and is now
closer to a different datacenter. In this situation, conflict avoidance breaks down, and
you have to deal with the possibility of concurrent writes on different leaders.

    - All replicas must arrive at the same final value
when all changes have been replicated, There are various ways of achieving **convergent conflict resolution**:

        - **Give each write a unique ID**(e.g., a timestamp, a long random number, a UUID,
or a hash of the key and value), If a timestamp is used, this technique is known
as last write wins (LWW). Although this approach is popular, it is dangerously
prone to data loss.

        - Give each replica a unique ID, and let writes that originated at a higher numbered replica always **take precedence over** writes that originated at a lower numbered replica. This approach also implies data loss

        - Somehow merge the values together—e.g., order them alphabetically and then
concatenate them

        - Record the conflict in an explicit data structure that preserves all information,
and write application code that resolves the conflict **at some later time**(perhaps
by prompting the user).


    - most appropriate way of resolving a conflict may depend on the application. On write: As soon as the database system detects a conflict in the log of replicated changes,
it calls the conflict handler(For example, Bucardo allows you to write a snippet of
Perl for this purpose); On read: When a conflict is detected, all the conflicting writes are stored. The next time
the data is read, these multiple versions of the data are returned to the applica‐
tion. The application may prompt the user or automatically resolve the conflict,
and write the result back to the database. (CouchDB works this way, for example).

    - For more info, see the **Automatic Conflict Resolution, ACR**(page 174). It includes several implementations of ACR. Likely that
they will be integrated into more replicated data systems in the future which make multi-leader data synchronization much simpler


    - Other kinds of conflict can be more subtle to detect. A conflict may arise if two different bookings are created
for the same room at the same time in meeting
room booking system. There can be a conflict if the two bookings are
made on two different leaders Even if the application checks availability before allowing a user to make a booking


- ### Multi-Leader Replication Topologies

    - (page 175 Figure 5-8.) The most general topology is all-to-all. MySQL by default supports only a circular topology (more restricted)  The fault tolerance of a more densely connected topology (such as
all-to-all) is better because it allows messages to travel along different paths, avoiding
a single point of failure.

        - A problem with circular and star topologies is that if just one node fails, it can interrupt the flow of replication messages between other nodes. Mostly,  reconfiguration would have to
be done manually

        - All-to-all topologies can have issues too.(Figure 5-9)This is a problem of causality, similar to the one we saw in “Consistent Prefix Reads”
on page 165: the update depends on the prior insert, so we need to make sure that all
nodes process the insert first, and then the update. To order these events correctly, a technique called version vectors can be used,

    - To prevent infinite replication loops, each node is given a unique
identifier, in the replication log, each write is tagged with the identifiers of all the nodes it has passed through. When a node receives a data change that is tagged with its own identifier, that data change is ignored, because the node knows that it has already been processed. 

    - Conflict detection techniques are poorly implemented in many multi-leader
replication systems

- ### Summary
    - 1. Why you want to replicate data?

    2. Synchronous and Asynchronous, Synchronous replication pros and cons

    3. Best practice for the replication on distributed database.

    4. The process of the Leader failure —— Failover, the concern of automatic failover

    5. Implementation of Replication Logs （Not quite clear）

    6. Solutions for Replication Lag 

    7. Use Cases for Multi-Leader Replication, Handling Write Conflicts

