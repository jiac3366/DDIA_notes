
## Chapter 4

- When a data format or schema changes, a corresponding change to application code often needs to happen. However, in a large application, code changes often cannot happen instantaneously, we need to maintain compatibility in both directions:
    - backend: Newer code can read data that was written by older code.
    - frontend: Older code can read data that was written by newer code.(Forward compati‐ bility can be trickier, because it requires older code to ignore additions made by a newer version of the code.)

- We will look at how those formats for encoding data, including JSON, XML, Protocol Buffers, Thrift, and Avro to handle schema changes and how they support systems where old and new data and code need to coexist
We will then discuss how those formats are used for data storage and for communication: in web services, Representational State Transfer (REST), and remote procedure calls (RPC), as well as message-passing systems such as actors and message queues

- ??? Figure 4-3  Figure 4-4. e Thrift CompactProtocol does this by packing the field type and tag number into a single byte, and by using variable-length integers. Rather than using a full eight bytes for the number 1337, it is encoded in two bytes, with the top bit of each byte used to indicate whether there are still more bytes to come. This means numbers between –64 and 63 are encoded in one byte, numbers between –8192 and 8191 are encoded in two bytes, etc. Bigger numbers use more bytes.  Finally, Protocol Buffers (which has only one binary encoding format) encodes the same data as shown in Figure 4-4. It does the bit packing slightly differently. but otherwise very similar to Thrift’s CompactProtocol. Protocol Buffers fits the same record in 33 bytes

- If old code (which doesn’t know about the new tag numbers you added) tries to read data written by new code, including a new field with a tag number it doesn’t recognize, it can simply ignore that field. The datatype annotation allows the parser to determine how many bytes it needs to skip. This maintains forward compatibility: **old code can read records that were written by new code。增加了新字段，老代码读到新代码写的数据可以无视掉它，这是老代码的向前兼容。**

- What about backward compatibility? As long as each field has a unique tag number, new code can always read old data, because the tag numbers still have the same meaning. **The only detail is that if you add a new field, you CANNOT make it required.** If you were to add a field and make it required, that check would fail if new code read data written by old code, because the old code will not have written the new field that you added. Therefore, to maintain backward compatibility, every field you add after the initial deployment of the schema must be optional or have a default value

- Removing a field is just like adding a field, with backward and forward compatibility concerns reversed. That means you can only remove a field that is optional (a required field can never be removed), and you can never use the same tag number again (because you may still have data written somewhere that includes the old tag number, and that field must be ignored by new code). 移除了字段，老的代码可能读到新代码写的数据缺失了字段，所以这个字段必须是非required的。这是新代码的向后兼容。

- A curious detail of Protocol Buffers is that it does not have a list or array datatype, but instead has a repeated marker for fields (which is a third option alongside required and optional). As you can see in Figure 4-4, the encoding of a repeated field is just what it says on the tin: **the same field tag simply appears multiple times in the record.** This has the nice effect that it’s okay to change an optional (single-valued) field into a repeated (multi-valued) field. New code reading old data sees a list with zero or one elements (depending on whether the field was present); old code reading new data sees only the last element of the list.


### Schema evolution rule

- Apache Avro is another binary encoding format that is interestingly different
from Protocol Buffers and Thrift. It was started in 2009 as a subproject of Hadoop, as
a result of Thrift not being a good fit for Hadoop’s use cases 

- With Avro, forward compatibility means that you can have a new version of the schema as writer and an old version of the schema as reader. Conversely, backward compatibility means that you can have a new version of the schema as reader and an old version as writer.
**To maintain compatibility, you may only add or remove a field that has a default value.** If you were to add a field that has no default value, new readers wouldn’t be able to read data written by old writers, so you would break backward compatibility. If you were to remove a field that has no default value, old readers wouldn’t be able to read data written by new writers, so you would break forward compatibility

### Dynamically generated schemas

- Avro: if the database schema changes (for example, a table has one column added and one column removed), you can just generate a new Avro schema from the updated database schema and export data in the new Avro schema. The data export process does not need to pay any attention to the schema change—it can simply do the schema conversion every time it runs.

- Thrift or Protocol Buffers: every time the database schema changes, an administrator would have to manually update the mapping from database column names to field tags. (It might be possible to automate this, but the schema generator would have to be very careful to not assign previously used field tags.)

- This kind of dynamically generated schema simply wasn’t a design goal of Thrift or Protocol Buffers, whereas it was for Avro.


### Code generation and dynamically typed languages ???

- Avro provides optional code generation for statically typed programming languages, but it can be used just as well **without any code generation**. you can simply open it using the Avro library and look at the data in the same way as you could look at a JSON file. The file is self-describing since it includes all the necessary metadata. This property is especially useful in conjunction with dynamically typed data processing languages like Apache Pig [26]. In Pig, you can just open some Avro files, start analyzing them, and write derived datasets to output files in Avro format without even thinking about schemas


### The Merits of Schemas

-  we can see that although textual data formats such as JSON, XML, and CSV are
widespread, binary encodings based on schemas are also a viable option. They have a
number of nice properties

    - They can be much more compact than the various “binary JSON” variants, since
they can omit field names from the encoded data.
    - The schema is a valuable form of documentation, and because the schema is
required for decoding, you can be sure that it is up to date (whereas manually
maintained documentation may easily diverge from reality).
    - Keeping a database of schemas allows you to check forward and backward com‐
patibility of schema changes, before anything is deployed.
    - For users of statically typed programming languages, the ability to generate code
from the schema is useful, since it enables type checking at compile time.


### Modes of Dataflow

-  Compatibility is a rela‐
tionship between one process that encodes the data, and another process that decodes
it. There are many ways data can flow from one process to
another. Who encodes the data, and who decodes it? some of the most common ways how data flows between processes:

    - ### Via databases (Dataflow Through Databases)
    
        - for the **data outlives code**: Most rela‐
tional databases allow simple schema changes, such as adding a new column with a
null default value, without rewriting existing data. When an old row is read, the
database fills in nulls for any columns that are missing from the encoded data on
disk

    - ### Via service calls (Dataflow Through Services: REST and RPC)

        - 

    - ### Via asynchronous message passing (Message-Passing Dataflow)

        - 
    