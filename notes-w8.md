
## Chapter 7 Transactions

- ### The Meaning of ACID

    - ACID was coined in an effort to
establish precise terminology for fault-tolerance mechanisms in databases. Let’s dig into the definitions of atomicity, consistency, isolation, and durability, as
this will let us refine our idea of transactions.

        - **Atomicity**,  in the context of ACID, atomicity does NOT
describe what happens if several processes try to access the same data at the same
time, because that is covered under the letter I, for **isolation**. Rather, Without atomicity, if an error occurs partway through making multiple changes, it’s
difficult to know which changes have taken effect and which haven’t. The application
could try again, but that risks making the same change twice, leading to duplicate or
incorrect data. Atomicity simplifies this problem: if a transaction was aborted, the
application can be sure that it didn’t change anything, so it can safely be retried. The ability to abort a transaction on error and have all writes from that transaction
discarded is the defining feature of ACID atomicity, Perhaps ***abortability*** would have
been a better term than atomicity, but we will stick with atomicity since that’s the
usual word

        - **Consistency**, this idea of consistency depends on the application’s notion of invariants,
and it’s the application’s responsibility to define its transactions correctly so that they
preserve consistency.  The application may rely
on the database’s atomicity and isolation properties in order to achieve consistency,
but it’s not up to the database alone. Thus, the letter C doesn’t really belong in ACID.  此处的一致性得从用户代码层面写好transaction的逻辑，数据的一致性才能得到保证

        - **Isolation**, Isolation in the sense of ACID means that concurrently executing transactions are
isolated from each other: they cannot step on each other’s toes,  in practice, serializable isolation is rarely used, because it carries a perfor‐
mance penalty,  We will explore snapshot isolation and other forms of isolation in “Weak Iso‐
lation Levels” on page 233

        - **Durability**,  is the promise that once a transaction has com‐
mitted successfully, any data it has written will not be forgotten. In practice, there is no one technique that can provide absolute guarantees. There are
only various risk-reduction techniques, including writing to disk, replicating to
remote machines, and backups—and they can and should be used together. As
always, it’s wise to take any theoretical “guarantees” with a healthy grain of salt.


- ### Weak Isolation Levels

	- ### Read Commited

		1. When reading from the database, you will only see data that has been committed (no dirty reads). 一个事务要更新几个对象时，防止其他事务看到中间态
		2. When writing to the database, you will only overwrite data that has been committed (no dirty writes). How to achieve: usually by delaying the second write until the first write’s transaction has
committed or aborted.

		- However, read committed does not prevent the race condition between two
counter increments in Figure 7-1. **In this case, the second write happens after the
first transaction has committed, so it’s not a dirty write.** It’s still incorrect, but for
a different reason—in “Preventing Lost Updates” on page 242 we will discuss how
to make such counter increments safe  所以脏写的概念是覆盖了未提交的写（脏读是读到了未提交的读），对于此处累加计数，虽然覆盖的是其他事务已经提交的写，但仍然没有办法保证计数器准确，需要用到其他办法。
	
		- ps: Some databases support an even weaker isolation level called read uncommitted. It prevents dirty writes (One client overwrites data that another client has written, but not yet committed.
Almost all transaction implementations prevent dirty writes),
but does not prevent dirty reads.

		- Implementing read committed: 
			- most databasesvi prevent dirty reads using the approach illustrated in
Figure 7-4: for every object that is written, the database remembers both the old com‐
mitted value and the new value set by the transaction that currently holds the write
lock. While the transaction is ongoing, any other transactions that read the object are
simply given the old value. Only when the new value is committed do transactions
switch over to reading the new value 对于read-only的请求, 防止脏读一般不用锁，因为一些写事务会极大影响它们。对于某个记录，数据库会记住当前修改其的写事务的旧值和新值，当写事务提交前，db返回旧值；写事务提交后，返回新值
			- 防止了脏读，也就防止了脏写? 不是这样的因果关系，预防脏写几乎所有隔离基本都能做到
			- 为什么RUC，没有防止脏读，却可以防止了脏写？这个可以去测试一下。

	- ### Snapshot Isolation and Repeatable Read

		-  The idea is that each transaction reads from a consistent snapshot of the database—that is, the transaction sees all the data that was committed in the database at the start of the transaction.

		- Implementing snapshot isolation
			- Like read committed isolation, implementations of snapshot isolation typically use write locks to prevent dirty writes. 
			
			- A key principle of snapshot isolation is readers
	never block writers, and writers never block readers. （但如果一些需要提前排他的资源，比如后面医生值班的例子，就需要读阻塞写） If a database only needed to provide read committed isolation, but not snapshot iso‐lation, it would be sufficient to keep two versions of an object: the committed version
	and the overwritten-but-not-yet-committed version. However, storage engines that
	support snapshot isolation typically use MVCC for their read committed isolation level as well. **A typical approach is that read committed uses a separate snapshot for
	each query, while snapshot isolation uses the same snapshot for an entire transaction.** MVCC通过在不同的时机开启新的snapshot，实现了RC和RR。When a transaction reads from the database, transaction IDs are used to decide which objects it can see and which are invisible. 正如Figure 7-7所形容的那样。

			- A long-running transaction may continue using a snapshot for a long time, continuing to read values that (from other transactions’ point of view) have long been over‐
	written or deleted. **By never updating values in place but instead creating a new
	version every time a value is changed**, the database can provide a consistent snapshot
	while incurring only a small overhead.

		- Indexes and snapshot isolation

			- PostgreSQL has optimizations for avoid‐
ing index updates if different versions of the same object can fit on the same page 对于发生事务的行（the same object can fit on the same page）对应的索引，不可以改变

			- Another approach： they use an append-only/copy-on-write variant
that does not overwrite pages of the tree when they are updated, but instead creates a
new copy of each modified page. However, this approach also requires a background pro‐
cess for compaction and garbage collection. ？

		- Repeatable read and naming confusion

			- Even though several databases implement repeatable read, there are big differences in the guarantees they actually provide, but most implementations don’t satisfy that formal definition
		
	- ### Preventing Lost Updates

		There are several interesting kinds of conflicts that can occur between concurrently writing transactions. we have only discussed dirty writes before.

		- ### Atomic write operations

			- Atomic operations are usually implemented by taking an exclusive lock on the object
when it is read so that no other transaction can read it until the update has been applied. This technique is sometimes known as cursor stability [36, 37]. Another
option is to simply force all atomic operations to be executed on a single thread.

		- ### Explicit locking

			- FOR UPDATE 行锁 排他锁，对于防止一些 read - modify - write cycle的逻辑出现线程安全问题，书中举了多个玩家移动像素，但不能同时移动同1个像素的例子。 

		- ### Automatically detecting lost updates

			- An advantage of this approach is that databases can perform this check efficiently in
conjunction with snapshot isolation. Indeed, PostgreSQL’s repeatable read, Oracle’s
serializable, and SQL Server’s snapshot isolation levels automatically detect when a
lost update has occurred and abort the offending transaction. However, MySQL/
InnoDB’s repeatable read does not detect lost updates [23].

		- ### Compare-and-set

			- If the content has changed and no longer matches 'old content', this update will
have no effect, so you need to check whether the update took effect and retry if neces‐
sary. 写入之前先比较当前版本是否是旧版本。However, if the database allows the WHERE clause to read from an old snapshot,
this statement may not prevent lost updates, because the condition may be true even
though another concurrent write is occurring. 而且确保该数据库where不是读的是旧的snapshot.

		- Ps: Locks and compare-and-set operations assume that there is a single up-to-date copy
of the data.However, databases with multi-leader or leaderless replication usually
allow several writes to happen concurrently and replicate them asynchronously, so
they cannot guarantee that there is a single up-to-date copy of the data.  (will revisit
this issue in more detail in “Linearizability” on page 324.) As discussed in “Detecting Concurrent Writes” , a common
approach in such replicated databases is to allow concurrent writes to create several
conflicting versions of a value (also known as siblings), 4and to use application code or
special data structures to resolve and merge these versions after the fact. 购物车例子. Unfortunately, LWW is the default in many replicated databases.


	- ### Write Skew and Phantoms

		- Figure 7-8. Show Example of write skew causing an application bug,  it is neither a dirty write nor a lost update 此案例是两名值班医生几乎同时想要取消自己值班，但违反了至少1名医生值班的规定（但在他们各自的事务看来，他们看到的值班表都有超过1名医生可以值班）
because the two transactions are updating two different objects (Alice’s and Bob’s on call records, respectively)  We can think of write skew as a generalization of the lost update problem. Write
skew can occur if two transactions read the same objects, and then update some of
those objects (different transactions may update different objects). In the special case
where different transactions update the same object, you get a dirty write or lost
update anomaly (depending on the timing).  there are various different ways of preventing lost updates. With write
skew, our options are more restricted: If you can’t use a serializable isolation level, the second-best option in this case is
probably to explicitly lock the rows that the transaction depends on 医生查询值班的记录时，要对返回的行进行加行级排他锁，因为后面有可能取消排班，所以让其他事务先别修改这一行（这是悲观锁，我觉得可以用乐观锁？）

		- Write skew Example2: Example 7-2. A meeting room booking system tries to avoid double-booking (not
safe under snapshot isolation) Unfortunately, snapshot isolation does not prevent another user from concur‐
rently inserting a conflicting meeting. In order to guarantee you won’t get scheduling conflicts, you once again need serializable isolation.  对于防止预定会议室时间上不能有冲突的这个场景，无法用FOR UPDATE去锁定行记录，因为记录还没有创建。

		- Example3: 还是游戏的例子，加锁没法防止2个玩家同时移动不同像素到同一个地方

		- 4：网站用户名唯一性，虽然快照隔离无法解决这个问题，但是数据库唯一约束可以做到。

		- 5: 钱包/库存问题，不能为负数。

	- ### Materializing conflicts 
	
		- to solve Phantoms causing write skew
方法：创建一个虚拟的表，充满了各种假的row, 然后锁定他们. Note that the additional table isn’t used to store information about the book‐ ing—it’s purely a collection of locks which is used to prevent bookings on the same room and time range from being modified concurrently。materializing conflicts should be considered a last resort if no alternative is possible. A serializable isolation level is much preferable in most cases (All along, the answer from researchers has been simple: **use serializable isolation**) 

