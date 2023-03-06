## Chapter1
- ### Three concerns that are important in most software systems

    - ### Reliability

        - Counter-intuitively, in such fault-tolerant systems, it can make sense to increase the
    rate of faults by triggering them deliberately.we generally prefer tolerating faults over preventing faults,there are cases
    where prevention is better than cure (e.g. because no cure exists, for example: if an attacker has compromised a system and
    gained access to sensitive data, that event cannot be undone )

        - Software errors is more important. Because of the data volumes and applications’ computing demands increase, more
    applications are using larger numbers of machine.Hence there is a move towards systems that can tolerate the loss of entire machines,
    by using software fault-tolerance techniques in preference to hardware redundancy

        - Human Error...

    - ### Scalability

        - Describing performance:(1) When you increase a load parameter, and keep the system resources (CPU,
    memory, network bandwidth, etc.) unchanged, how is performance of your sys‐
    tem affected? (2) When you increase a load parameter, how much do you need to increase the
    resources if you want to keep performance unchanged?

        - for the batch processing:

            - throughput, the number of records we can process per second
            - cost time, the total time it takes to run a
    job on a dataset of a certain size. 

        - for the online service:
            
            - response time of a service is usually more important
            
            - Most requests are reasonably fast, but there are occasional outliers that take much longer. Perhaps the slow requests are intrinsically more expensive, e.g. because they process more data, But even in a scenario where you’d
    think all requests should take the same time, you get variation: random additional
    latency could be introduced by a context switch to a background process, the loss of a
    network packet and TCP retransmission, a garbage collection pause, a page fault
    forcing a read from disk, mechanical vibrations in the server rack, or many other
    things.

            - to better tell you how many users actually experienced that delay, use percentiles instead average. eg: if median response time is 200 ms, that means half your requests return in less than
    200 ms, and half your requests take longer than that. The median is also known as 50th
    percentile (sometimes abbreviated as p50). For example, if the 95th percentile response time
    is 1.5 seconds, that means 95 out of 100 requests take less than 1.5 seconds, and 5 out
    of 100 requests take 1.5 seconds or more

            - SLA(service level agreements):  SLA may state that the service is considered to be up if it has a
    median response time of less than 200 ms and a 99th percentile under 1 s (if the
    response time is longer, it might as well be down), and the service may be required to
    be up at least 99.9% of the time.

            -  it only takes a small number of slow requests to hold up the processing of subsequent requests — an effect sometimes known as head-of-line

        - generating load artificially in order to test the scalability of a system, the loadgenerating client needs to keep sending requests independently of the response time.
    If the client waits for the previous request to complete before sending the next one,
    that behavior has the effect of artificially keeping the queues shorter in the test than
    they would be in reality, which skews the measurements blocking

    - ### Maintainability
    
        -  three design principles for software systems

            - Operability

            - Simplicity

            - Evolvability

---

## Chapter2 

- ### Difference between One2Many & Many2One

    - **How to represente one-to-many relationship**:
    
        - 1. Foreign key. 
        - 2. Later versions of the SQL standard added support for structured datatypes and
        XML data, which allow multi-valued data to be stored within a single row, with
        support for querying and indexing inside those documents. ?PostgreSQL also has vendor-specific extensions for JSON, array
        datatypes
        - 3. Encode as a JSON or XML document(typically cannot use the database to query for val‐
    ues inside that encoded column) (Chapter 4,
    there are also problems with JSON as a data encoding format.)

    - **The advantage of using an ID(Foreign key in relationship db)** Because it has no meaning to humans. Anything that is meaningful to humans may need to change sometime in
    future. If that information is duplicated, all the redundant copies need to be
    updated. That incurs overhead on writes, and risks inconsistencies.

    -  PS: query optimizer is the role of deciding the access paths for the query.  the query optimizer automatically decides which parts of the
    query to execute in which order, and which indexes to use. not by the application developer, so we rarely need to think
    about them.

    - relational and document databases are not fundamentally different on representing many-to-one and many-to-many relation‐
    ships —— the related item is referenced by a unique identifier, which is called a foreign
    key in the relational model, and a document reference in the document model

- ### Relational vs Document databases 

    - if there is Many2many relationship, using a document model can lead to significantly more complex
application code and worse performance

    - ps: For highly
interconnected data, the document model is very awkward, the relational model is
acceptable, and graph models (see “Graph-like Data Models” on page 48) are the most
natural.

    - **A more accurate term is
schema-on-read (the structure of the data is implicit, and only interpreted when the
data is read), in contrast with schema-on-write (the traditional approach of relational
databases, where the schema is explicit and the database ensures all data conforms to
it)**

- ### Query Languages for Data

    - An imperative language tells the computer to perform certain operations in a certain
order; A declarative query language hides imple‐
mentation details of the database engine, which makes it possible for the database
system to introduce performance improvements without requiring any changes to
queries.

    - Web browser example: CSS selector(declarative) and JS(imperative)




### Question: 

- updates to a document, the entire document usually needs to be re-written?



