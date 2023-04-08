
- ### Serializability
	Define: To execute only one transaction at a time, in serial order, on a single thread

	- Most databases that provide serializability today use one of three techniques：

		- Literally executing transactions in a serial order

		- Two-phase locking

		- Optimistic concurrency control techniques such as serializable snapshot isolation

	- If multi-threaded concurrency was considered essential for getting good performance during the previous 30 years, what changed to make single-threaded execution possible?

		- RAM便宜了，内存大，事务要处理的数据可能都会在内存中处理

		- OLTP数据库的事务都很短，并且涉及的是少量的读写

	- ### 1.Actual Serial Execution

		- 在一些订票的场景，用户会有很多阶段（选路线，选座位，付钱），但n, systems with single-threaded serial transaction processing don’t allow interactive multi-statement transactions. Instead, the application must submit the entire transaction code to the database ahead of time, as a stored procedure. Figure 7-9. The difference between an interactive transaction and a stored procedure (using the example transaction of Figure 7-8).

		- Modern implementations of stored proce‐ dures have abandoned PL/SQL and use existing general-purpose programming lan‐ guages instead: VoltDB uses Java or Groovy, Datomic uses Java or Clojure, and Redis uses Lua. With stored procedures and in-memory data, executing all transactions on a single thread becomes feasible. As they don’t need to wait for I/O and they avoid the over‐ head of other concurrency control mechanisms, they can achieve quite good throughput on a single thread

		- Summary of serial execution

			- 每个事务都必须小而快

			- 它仅限于活动数据集可以放入内存的用例。 很少访问的数据可能会移动到磁盘，但如果需要在单线程事务中访问它，系统会变得非常慢
			
			- 写入吞吐量必须足够低以在单个CPU 内核上处理，否则需要对事务进行分区而不需要跨分区协调。

			- 跨分区事务是可能的，但它们的使用范围有硬性限制

	- ### 2.Two-Phase Locking (2PL)

		- In 2PL, writers don’t just block other writers; they also block readers and vice versa. Snapshot isolation has the mantra readers never block writers, and writers never block readers (see “Implementing snapshot isolation” on page 239), which captures this key difference between snapshot isolation and two-phase locking.

		- ### Implementation of two-phase locking

			- 2PL is used by the serializable isolation level in MySQL (InnoDB) and SQL Server, and the repeatable read isolation level in DB2 [23, 36]

			- 读的时候是使用共享锁，写的时候用排他锁，读变成写也会有锁升级

			- This is where the name “two phase” comes from: the first phase (while the transaction is executing) is when the locks are acquired, and the second phase (at the end of the transaction) is when all the locks are released. So deadlock: The database automatically detects deadlocks between transactions and aborts one of them so that the others can make progress. The aborted transaction needs to be retried by the application.
		
		- ### Predicate locks

			- If transaction A wants to read objects matching some condition, like in that
SELECT query, it must acquire a shared-mode predicate lock on the conditions of
the query
			- If transaction A wants to insert, update, or delete any object, it must first check
whether either the old or the new value matches any existing predicate lock.

			- The key idea here is that a predicate lock applies even to objects that do not yet exist
in the database, but which might be added in the future (phantoms). If two-phase
locking includes predicate locks, the database prevents all forms of write skew and
other race conditions, and so its isolation becomes serializable


			- Unfortunately, predicate locks do not perform well: if there are many locks by active
transactions, checking for matching locks becomes time-consuming. For that reason,
most databases with 2PL actually implement index-range locking (also known as next key locking), which is a simplified approximation of predicate locking

		- ### Index-range locks

			- 不锁具体的行，锁符合条件范围的索引项，甚至锁的范围可以更加广。 If you have a predicate lock for bookings of room 123 between noon and 1 p.m.,
you can approximate it by locking bookings for room 123 at any time, or you can
approximate it by locking all rooms (not just room 123) between noon and 1 p.m

			- Index-range
locks are not as precise as predicate locks would be (they may lock a bigger range of
objects than is strictly necessary to maintain serializability), but since they have much
lower overheads, they are a good compromise.If there is no suitable index where a range lock can be attached, the database can fall
back to a shared lock on the entire table????. This will not be good for performance, since
it will stop all other transactions writing to the table, but it’s a safe fallback position.



	- ### 3.Serializable Snapshot Isolation (SSI)

		- 2008才被发表的算法，可以兼顾serializable isolation and good perfor‐
mance？ it has the possibility of being
fast enough to become the new default in the future

		- 对于2阶段锁，都是悲观的并发控制，因为其认为it’s better to wait until the situation is safe again before doing anything。对于序列化执行，就是极度悲观。

		- SSI is based on snapshot isolation—that is, all reads within a
transaction are made from a consistent snapshot of the database  This is the main difference compared to earlier optimistic concurrency control techniques（compare & set）. On top of snapshot isolation, SSI adds an algorithm for detecting serialization conflicts among writes and determining
which transactions to abort. In other words, in order to provide serializable isolation, the database must detect situations in which a transaction may have acted on an outdated premise and abort the transaction in that case (避免snapshot isolation的写动作基于了一个过期的premise)

		- How does the database know if a query result might have changed? There are two cases to consider：

			- 1.Detecting stale MVCC reads (uncommitted write occurred before the read)
			Detecting reads of a stale MVCC object version， SSI preserves snapshot isolation’s support for long-running reads from a consistent snapshot。在本事务即将要提交的时候，数据库检查是否存在【被无视的写】（即本事务开启的时候，本事务依赖的数据是否存在一些未提交的写）已经被提交了，如果有，则放弃本事务。ssi开一个线程不断读snapshot???

			- 2.Detecting writes that affect prior reads
			Detecting writes that affect prior reads (the write occurs after the read) 在其他事务写了本事务依赖的数据的时候，可以通过在index上面记录一些信息，达到通知本事务read outdated的作用？
		
		- Performance of serializable snapshot isolation

			- one trade-off is the granularity at which transactions’ reads and writes
are tracked. If the database keeps track of each transaction’s activity in great detail, it
can be precise about which transactions need to abort, but the bookkeeping overhead
can become significant. Less detailed tracking is faster, but may lead to more transac‐
tions being aborted than strictly necessary. 事务粒度小，性能差

			- SSI继承了snapshot isolation的好处，读写互不阻塞。感觉就是CA S的思想

			- serializable snapshot isolation is not limited to the
throughput of a single CPU core: FoundationDB distributes the detection of seriali‐
zation conflicts across multiple machines, allowing it to scale to very high throughput. ？

			- SSI requires that read-write transactions be fairly short (long-running read-only transactions may be okay). However, SSI is probably less sensitive
to slow transactions than two-phase locking or serial execution.


- ###Summary

    - good!

    - The examples in this chapter used a relational data model. However, as discussed in
“The need for multi-object transactions” on page 231, transactions are a valuable
database feature, no matter which data model is used.
In this chapter, we explored ideas and algorithms mostly in the context of a database
running on a single machine. Transactions in distributed databases open a new set of
difficult challenges, which we’ll discuss in the next two chapters.