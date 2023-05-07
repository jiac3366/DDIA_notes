
## Chapter10 Batch Processing

- ### Batch Processing with Unix Tools

    - urprisingly many data analyses can be done in a few
minutes using some combination of awk, sed, grep, sort, uniq, and xargs, and they
perform surprisingly well

    - ### Sorting versus in-memory aggregation 

        - ! 一般我们写代码都是创建的一个Counter Hash Table, key是url， value是number; 但是The Unix pipeline example does not have
such a hash table, but instead relies on sorting a list of URLs in which multiple occurrences of the same URL are simply repeated.对URL列表进行排序，然后统计。哪种更好？
        - 一般如果数据量比内存小，可以直接在内存中用。但如果数据量比内存大，可以借用SSTable的思想（Unix pipeline的方法）部分的数据可以在内存中排序（借用AVL或者红黑树），超过一定阈值就写到内存中形成小文件，再用归并排序合并多个排序好的小文件。The bottleneck
is likely to be the rate at which the input file can be read from disk.

    - ### The Unix Philosophy. 
        the idea of connecting programs with pipes became part of what is now known as the Unix philosophy。 This approach—automation, rapid prototyping, incremental iteration, being friendly
to experimentation, and breaking down large projects into manageable chunks—
sounds remarkably like the Agile and DevOps movements of today.  Even though many of these programs are written by
different groups of people, they can be joined together in flexible ways. What does
Unix do to enable this composability?


        - A uniform interface
        
            it’s actually quite remarkable that these very
different things can share a uniform interface, so they can easily be plugged together

        - Separation of logic and wiring


        - Transparency and experimentation

            Part of what makes Unix tools so successful is that they make it quite easy to see what is going on

            - You can end the pipeline at any point, pipe the output into less, and look at it to see if it has the expected form. This ability to inspect is great for debugging

            - However, the biggest limitation of Unix tools is that they run only on a single machine—and that’s where tools like Hadoop come in

    
- ### MapReduce and Distributed Filesystems

    - A single MapReduce job is comparable to a single Unix process: it takes one or more inputs and produces one or more outputs
    - As with most Unix tools, running a MapReduce job normally does not modify the input and does not have any side effects other than producing the output.
    - In this chapter we will mostly use HDFS as a running example, but the principles apply to any dis‐ tributed filesystem. One difference is that with HDFS, computing tasks can be scheduled to run on the machine that stores a copy of a particular file, whereas object stores (Amazon S3) usually keep storage and computation separate. Reading from a local disk has a performance advantage if network bandwidth is a bottleneck. Note however that if erasure coding is used, the locality advantage is lost, because the data from several machines must be combined in order to reconstitute the original file.
    - HDFS is based on the shared-nothing principle (see the introduction to Part II), in contrast to the shared-disk approach of Network Attached Storage (NAS) and Storage Area Network (SAN) architectures. Shared-disk storage is implemented by a central‐ ized storage appliance, often using custom hardware and special network infrastruc‐ ture such as Fibre Channel. On the other hand, the shared-nothing approach requires no special hardware, only computers connected by a conventional datacenter network.


    - ### MapReduce Job Execution

        - python counter = "sort | uniq -c" in linux，感觉extract + sort才算是一个单机mapper要做的全部事情

        - To create a MapReduce job, you need to implement two callback functions, the mapper and reducer, which behave as follows:

            - Mapper: The mapper is called once for every input record Mapper会应用于每条record, and its job is to extract the key
and value from the input record. For each input, it may generate any number of key-value pairs (including none). 

            - Reducer: The MapReduce framework takes the key-value pairs produced by the mappers,
collects all the values belonging to the same key, and calls the reducer with an
iterator over that collection of values. 对Mapper生成的相同的key-value pair(groupby 组)应用相同的Reducer（聚合函数)

        - Figure 10-1 shows the dataflow in a Hadoop MapReduce job. Its parallelization is
based on partitioning (see Chapter 6): the input to a job is typically a directory in
HDFS, and each file or file block within the input directory is considered to be a sepa‐
rate partition that can be processed by a separate map task

        - MapReduce Process: 
            - While the number of map
tasks is determined by the number of input file blocks, the number of reduce tasks is
configured by the job author (it can be different from the number of map tasks). To
ensure that all key-value pairs with the same key end up at the same reducer, the
framework uses a hash of the key to determine which reduce task should receive a
particular key-value pair 在Mapper端的多台机器会有同一个groupby key的组，需要经历shuffle到同一台机器的Reducer上。Each of these partitions is written to a sorted file on the mapper’s
local disk, using a technique similar to what we discussed in “SSTables and LSM Trees” on page 76. 每个Mapper机器排序后的相同key的groupby组（排序好的小文件）如何合并成一个Reducer端的一台机器上呢？（大文件）其原理就是SSTables and LSM Trees

            - Whenever a mapper finishes reading its input file and writing its sorted output files,
the MapReduce scheduler notifies the reducers that they can start fetching the output
files from that mapper. The reducers connect to each of the mappers and download
the files of sorted key-value pairs for their partition. 当mapper任务完成，reducer会去mapper的sorted file中下载其对应的groupby组。The process of partitioning by
reducer, sorting, and copying data partitions from mappers to reducers is known as
the **shuffle**

        - The reduce task takes the files from the mappers and merges them together, preserving the sort order. Thus, if different mappers produced records with the same key, they will be adjacent in the merged reducer input.

        - ### MapReduce workflows

            - Chained MapReduce jobs are therefore less like pipelines of Unix commands (which
pass the output of one process as input to another process directly, using only a small
in-memory buffer) and more like a sequence of commands where each command’s
output is written to a temporary file, and the next command reads from the tempo‐
rary file.

            - various workflow schedulers for Hadoop have been developed, including
Oozie, Azkaban, Luigi, Airflow, and Pinball。 Various higher-level tools for Hadoop, such as Pig [30], Hive [31], Cascading [32],
Crunch [33], and FlumeJava [34], also set up workflows of multiple MapReduce
stages that are automatically wired together appropriately. Workflows consisting of 50 to 100 MapReduce jobs are
common when building recommendation systems

    - ### Reduce-Side Joins and Grouping

        - ### Sort-merge joins

            -  Figure 10-3 分析用户日志行为，问题是： Let system could
        determine which pages are most popular with which age groups. A better approach would be to take a copy of the user database (for example,
        extracted from a database backup using an ETL processsee “Data Warehousing” ) and to put it in the same distributed filesystem as the log of user activity
        events. You would then have the user database in one set of files in HDFS and the
        user activity records in another set of files, and could use MapReduce to bring
        together all of the relevant records in the same place and process them efficiently

                - 一个mapper任务就像Figure 10-2例子一样，从中抽取出{userID: url}; 另一个mapper任务从用户DB数据中抽取出{userID: date of birth}

                - 然后reducer可以根据userID的奇偶性，来定义分区键。并且排序多个mapper的output file （**secondary sort**）, 就把userID想同的数据给临近在一起了

                - 一些中间性总结：which pages are most popular with which age groups。书中的目的就是弄成这种形式：{url: [(username1: 1991), (username2: 1992), ...]}  从而Subsequent MapReduce jobs could then calculate the distribution of viewer ages for each URL, and cluster by age group（就如用户的R值一样，给用户分去不同的价值组，在这里，就是给url分不同的年龄爱好组？）.

                - ！从一堆埋点数据中，给key(user / url)做聚合（计算RFM / 拼接url对应的用户profile lists)。 根据”agg值“训练聚合模型，从而利用聚合模型，对”agg值“进行分类

                - Since the reducer processes all of the records for a particular user ID in one go, it only
        needs to keep one user record in memory at any one time, and it never needs to make
        any requests over the network. This algorithm is known as a sort-merge join, **since
        mapper output is sorted by key, and the reducers then merge together the sorted lists
        of records from both sides of the join**？？？


            - Using the MapReduce programming model has separated the physical network com‐
munication aspects of the computation (getting the data to the right machine) from
the application logic (processing the data once you have it). This separation contrasts
with the typical use of databases, where a request to fetch data from a database often
occurs somewhere deep inside a piece of application code. MapReduce计算框架已经使得计算逻辑与网络问题分隔开了

        - ### Bringing related data together in the same place & Handling skew

            - MapReduce在用户日志分析中的应用： If you have multiple web servers handling user requests, the activity events for a par‐
ticular user are most likely scattered across various different servers’ log files. You can
implement sessionization by using a session cookie, user ID, or similar identifier as
the grouping key and bringing all the activity events for a particular user together in
one place, while distributing different users’ events across different partitions. 
精髓就是：bringing all records with the same key to the same place

            - 但上述精髓会存在一些热点倾斜问题，比如一位百万up主，它的key对应的数据就是海量这么多。 a MapReduce job is only
complete when all of its mappers and reducers have completed, any subsequent jobs
must wait for the slowest reducer to complete before they can start。应对方法是先检测出热点key（手动指定或者using a sampling job)，让热点key的groupby组分配到多个reducer处理(与page 205的 using randomization to alleviate hot spots in a partitioned data‐
base类似)


    - ### Map-Side Joins

        - 只有reduce-side join的缺点： the downside is that all that
sorting, copying to reducers, and merging of reducer inputs can be quite expensive.
Depending on the available memory buffers, data may be written to disk several
times as it passes through the stages of MapReduce


        -  It is
possible to make joins faster by using a so-called map-side join. This approach uses a
cut-down MapReduce job in which there are no reducers and no sorting. Instead,
each mapper simply reads one input file block from the distributed filesystem and
writes one output file to the filesystem—that is all.


















