
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
machine running the database software is called a node. Each node uses its CPUs,
RAM, and disks independently. Any coordination between nodes is done at the soft‐
ware level, using a conventional network. No special hardware is required by a shared-nothing system. In this part of the book, we focus on shared-nothing architectures because they require the
most caution from you, the application developer.  But first, let’s talk about distributed data

## Chapter 5 Replication

- again why you want to replicate data:

    - To keep data geographically close to your users (and thus reduce latency)
    - To allow the system to continue working even if some of its parts have failed
(and thus increase availability)
    - To scale out the number of machines that can serve read queries (and thus
increase read throughput)


- ### Leaders and Followers

    - leader-based replication
is not restricted to only databases: distributed message brokers such as Kafka [5] and
RabbitMQ highly available queues [6] also use it. Some network filesystems and
replicated block devices such as DRBD are similar.

    -  There are circumstances when followers might fall behind the leader by several
minutes or more; for example, if a follower is recovering from a failure, if the system
is operating near maximum capacity, or if there are network problems between the
nodes

    - synchronous replication pros and cons:

        - guaranteed to have
an up-to-date copy of the data that is consistent with the leader. If the leader sud‐
denly fails, we can be sure that the data is still available on the follower.

        - The disadvantage is that if the synchronous follower doesn’t respond, the write cannot be processed.
The leader must block all writes and wait until the synchronous replica is available
again

    - In practice, if you enable syn‐
chronous replication on a database, it usually means that one of the followers is syn‐
chronous, and the others are asynchronous. If the synchronous follower becomes
unavailable or slow, one of the asynchronous followers is made synchronous. This
guarantees that you have an up-to-date copy of the data on at least two nodes: leader and one synchronous follower (semi-synchronous)

    - But Oftenly, leader-based replication is configured to be completely asynchronous especially if there are many followers or if they are geographically distributed. Even though a write is not guaranteed to be durable( any writes that have not yet been repli‐
cated to followers are lost)

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
from its log, it knows the last transaction that was processed before the fault occur‐
red.  the follower can connect to the leader and request all the data changes that
occurred during the time when the follower was disconnected

    - **Leader failure: Failover**: Handling a failure of the leader is trickier: one of the followers needs to be promoted
to be the new leader,  clients need to be reconfigured to send their writes to the new
leader, and the other followers need to start consuming data changes from the new
leader. This process is called **failover**. 故障切换

        - Failover can happen manually (an administrator is notified that the leader has failed
and takes the necessary steps to make a new leader)

        - An automatic
failover process usually consists of the following steps：

            1. Determining that the leader has failed， if a node doesn’t respond for some period of time—say, 30 seconds—it is
    assumed to be dead

            2. Choosing a new leader. Could be done through an election process (chosen by  majority of the remaining replicas )or be appointed by a previously elected **controller node**???  The best candidate for
    leadership is usually the replica with the most up-to-date data changes from the
    old leader. Getting all the nodes to agree on a new
    leader is a consensus problem, discussed in detail in Chapter 9.

            3. Reconfiguring the system to use the new leader. . If the old leader comes back, it might still believe that it is the leader,
    not realizing that the other replicas have forced it to step down. The system
    needs to ensure that the old leader becomes a follower and recognizes the new
    leader

        - Failover is fraught with things that can go wrong:

            1. If asynchronous replication is used, the new leader may not have received all the
writes from the old leader before it failed.  The most common
solution is for the old leader’s unreplicated writes to simply be discarded, which
may violate clients’ durability expectations. ?

            2. Discarding writes is especially dangerous if other storage systems outside of the
database need to be coordinated with the database contents. eg: one incident at Github, the new leader’s counter lagged behind the old leader’s, it
reused some primary keys that were previously assigned by the old leader. These
primary keys were also used in a Redis store, so the reuse of primary keys resul‐
ted in inconsistency between MySQL and Redis

            3. likely happen that two nodes both
believe that they are the leader. This situation is called **split brain**, and it is dan‐
gerous: if both leaders accept writes, and there is no process for resolving conflicts (see “Multi-Leader Replication” on page 168),data is likely to be lost or
corrupted. As a safety catch, some systems have a mechanism to shut down one
node if two leaders are detected.ii However, if this mechanism is not carefully
designed, you can end up with both nodes being shut down

            4. If the system is already strug‐
gling with high load or network problems, an unnecessary failover is likely to
make the situation worse, not better. System should define the right timeout before the leader is declared dead —— A longer timeout
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