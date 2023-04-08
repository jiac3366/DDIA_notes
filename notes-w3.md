
## Chapter2

- ### MapReduce querying

    - MapReduce is a fairly low-level programming model for distributed execution on a
cluster of machines. Higher-level query languages like SQL can be implemented as a
pipeline of MapReduce operations


- ### Graph-like Data Models

    -  Graphs are not limited
to such homogeneous data: an equally powerful use of graphs is to **provide a consis‐
tent way of storing completely different types of object in a single data store**. 

    - PS: For
example, Facebook maintains a single graph with many different types of vertex and
edge: vertices represent people, locations, events, checkins and comments made by
users; edges indicate which people are friends with each other, which checkin hap‐
pened in which location, who commented on which post, who attended which event,
etc.

    -  **Tool**: The property graph model (implemented by
Neo4j, Titan, InfiniteGraph) and the triple-store model (implemented by Datomic,
AllegroGraph and others).

    - **Languag**e: Three declarative query languages for
graphs: Cypher, SPARQL, and Datalog. Besides these, there are also imperative graph
query languages such as Gremlin and graph processing frameworks like Pregel

- ### Query language of Graph

    - The **Cypher** vs SQL query in graph model data, declarative query language 

        - Example 2-3 shows the Cypher query to insert the left-hand portion of Figure 2-5
    into a graph database. Example 2-4. Cypher query to find people who emigrated from the US to Europe. 

        - We also can query the vertex via SQL in graph which stored in relational database. But  In a relational database, you usually know
    in advance which joins you need in your query. In a graph query, you may need to traverse a variable number of edges before you find the vertex—
    that is, **the number of joins is not fixed in advance**. Moreover, the SQL syntax is
    very clumsy in comparison to Cypher. (sup‐
    ported in PostgreSQL, IBM DB2, Oracle, and SQL Server)

            - Find all vertex that whthin USA

            - Find all person which "BORN_IN" USA

            - same like finding all person "LIVE_IN" EUR

    - Triple-Stores and SPARQL

        - In a triple-store, all information is stored in the form of very simple three-part state‐
ments: (subject, predicate, object)

        - The triple-store data model is completely independ‐
ent of the semantic web—for example, Datomic is a triple-store that does not
claim to have anything to do with it

        - SPARQL is a nice query language ——— even if the semantic web never happens, it can be
a powerful tool for applications to use internally. It is even more concise in SPARQL than it is in Cypher.

        - Graph Databases Compared to the Network Model, some disscion about CODASYL’s network model and graph model.graph model is better.

    - The Foundation: Datalog

        - Datalog is a much older language than SPARQL or Cypher,Datalog is used in a few data systems: for example, it is the query language of Datomic, and Cascalog is a Datalog implementation for querying
large datasets in Hadoop

        - Datalog principle kind of likes the recursion query that one function can be called by other function.




- ### Summary

    - developers found
that some applications don’t fit well in the relational model either. New non relational “NoSQL” data stores have diverged in two main directions:
        
        - Document databases target use cases where data comes in self-contained documents, and relationships between one document and another are rare

        - Graph databases go in the opposite direction, targeting use cases where anything
is potentially related to everything

        - One thing that document and graph databases have in common is that they typically
don’t enforce a schema for the data they store, which can make it easier to adapt
applications to changing requirements.

    - Other database software bases on the specific data models:
    
        - Researchers working with genome data often need to perform sequence similarity searches

        - Particle physicists have been doing Big Data-style large scale data analysis for
decades

        - Full-text search is arguably a kind of data model that is frequently used alongside
databases.

---

## Chapter3

- ### Hash Index

    - Every key is mapped to a byte offset in the data file—the location at
which the value can be found, Whenever you append a
new key-value pair to the file, you also update the hash map to reflect the offset of the
data you just wrote( Figure 3-1 ) 

    - eg: This is essentially what
Bitcask (the default storage engine in Riak) does. A storage engine like Bitcask is well suited to situations where the value for each key
is updated frequently. For example, the key might be the URL of a cat video, and the
value might be the number of times it has been played. But how do we avoid eventually
running out of disk space —— We can then perform compaction on these
segments, as illustrated in Figure 3-2，retaining only the most recent value for each key. We can also merge
several segments together at the same time as performing the compaction, as shown
in Figure 3-3.

    - **The whole process of mergeing segment**: The merging and compaction of frozen seg‐
ments can be done in a background thread, and while it is going on, we can still continue to serve read and write requests as normal, using the old segment files. After the
merging process is complete, we switch read requests to using the new merged segment instead of the old segments—and then the old segment files can simply be
deleted

    - Speical point of implementation:

        - Deleting records, When log seg‐
ments are merged, the tombstone tells the merging process to discard any previ‐
ous values for the deleted key.

        - Crash recovery, server restarts painful cause it might take a long time if the segment files are large,
which would make server restarts painful. Bitcask speeds up recovery by storinga snapshot of each segment’s hash map on disk, which can be loaded into mem‐
ory more quickly.

        - Partially written records, server may crash at any time, including halfway through appending a
record to the log. Bitcask files include checksums, allowing such corrupted parts
of the log to be detected and ignored

        - Concurrency control, have only one writer thread. Data file segments are
append-only and otherwise immutable

    - Advantage of Append-only design:

        - Concurrency and crash recovery are much simpler

        - sequential write operations, which are generally much faster than random writes

    - Disadvantage of hash table index:

         - Bad perfomance if you have to maintain a hash map on disk cause it
requires a lot of random access I/O,

         - Range queries are not efficient on hash table.





    
