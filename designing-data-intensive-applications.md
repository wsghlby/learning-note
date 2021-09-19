# Designing Data-Intensive Applications

[Chapter 1 Reliable, Scalable, and Maintainable Applications](#chapter-1-reliable-scalable-and-maintainable-application)
- [Reliability](#reliability)
- [Scalability](#scalability)
- [Maintainability](#maintainability)

[Chapter 2 Data Models and Query Languages](#chapter-2-data-models-and-query-languages)
- [Relational model versus document model](#relational-model-versus-document-model)
- [Query language](#query-language)
- [Graph-like data models](#graph-like-data-models)

[Chapter　3 Storage and Retrieval](#chapter-3-storage-and-retrieval)
- [Data Structures That Power Your Database](#data-structures-that-power-yourd-atabase)
- [Database for Data Analytics](#database-for-data-analytics)
- [Column-Oriented Storage](#column-oriented-storage)


[Chapter 4 Encoding and Evolution](#chapter-4-encoding-and-evolution)
- [Formats for Encoding Data](#formats-for-encoding-data)
- [Modes of Dataflow](#modes-of-dataflow)

[Chapter 5 Replication](#chapter-5-replication)
- [Leaders and Followers](#leaders-and-followers)
- [Problems with Replication Lag](#problems-with-replication-lag)
- [Multi-Leader Replication](#multi-leader-replication)
- [Leaderless Replication](#leaderless-replication)

# Chapter 1 Reliable, Scalable, and Maintainable Applications

Data-intensive applications, as opposed to compute-intensive ones, worry more about data - the amount of data, the complexity of data, and the data changing speed.

Functionality data-intensive applications commonly need:
- database: store data somewhere so that it could be retrieved in the future
- cache: remember and reuse the operation result instead of spending time to repeat the execution
- search index: all user to search data by keyword or filter it in various ways
- stream processing: send a message to another process, to be handled asynchronously
- batch processing: periodically crunch a large amount of accumulated data

This chapter is about exploring ways of thinking about reliability, scalability, and maintainability.

## Reliability

The system should continue work correctly even in the face of fault.

### Typical expectation

- The application performs the function that the user expected.
- Its performance is good enough for the required use case, under the expected load and data volume.
- It can tolerate the user making mistakes or using the software in unexpected ways.
- The system prevents any unauthorized access and abuse.

### Fault vs. Failure

The things that can go wrong are called faults, and systems that anticipate faults and can cope with them are called fault-tolerant or resilient. Failure is when the system as a whole stops providing the required service to the user. 

### Hardware faults

It's common to have hardware faults when you have a lot of machines. 2 ways to deal with it:
1. Add redundancy to individual hardware (e.g. dual power supplies). When one component dies, the redundant component can take its place.
2. Use software fault-tolerance techniques to tolerate the loss of entire machines.

### Software errors

Such faults are harder to anticipate, and because they are correlated across nodes, they tend to cause many more system failures than uncorrelated hardware faults.

The reason is that software is making some kind of assumption about its environment which may stop being true for some reason.

Some small things can help: carefully thinking about assumptions and interactions in the system; thorough testing; process isolation; allowing processes to crash and restart; measuring, monitoring, and analyzing system behavior in production; constantly check assumption the software relay on.

### Human errors

Humans are known to be unreliable. Approaches that make our systems reliable, in spite of unreliable humans:
- Design systems in a way that minimizes opportunities for error. (e.g. well-designed API).
- Decouple the places where people make the most mistakes from the places where they can cause failures (e.g. sandbox where people can explore and experiment safely, without affecting real users).
- Allow quick and easy recovery from human errors, to minimize the impact in the case of a failure.
- Set up detailed and clear monitoring which can show us early warning signals and allow us to check whether any assumptions or constraints are being violated.
- Implement good management practices and training.
- Test thoroughly at all levels, from unit tests to whole-system integration tests and manual tests.

## Scalability

Scalability is the term we use to describe a system’s ability to cope with increased load (having strategies for keeping performance good, even when load increases).

### How to describe system load

Load can be described with a few numbers which we call load parameters. The best choice of parameters depends on the architecture of your system (e.g. request per second for web server, ratio of reads to writes for database; average case, extreme cases).

#### Twitter example

2 main operations:
- Post tweet  to this user's followers (4.6k requests/sec on average, over 12k requests/sec at peak).
- Home timeline display tweets posted by people this user follow (300k requests/sec).

2 ways of implementing these operations:
1. Store tweet in a global collection. When a user request their timeline, retrieve their follows' tweets from the tweet collection and sort.
2. Maintain a cache for each user's home timeline. When a user post a tweet, update all their follower's timeline cache.

Approach 1 make Twitter struggled with the load of home timeline queries, so it switched to approach 2. This works better because average tweet posting rate is much lower than home timeline requesting rate, so in this case it’s preferable to do more work at write time and less at read time.

However, the downside of approach 2 is that posting a tweet now requires a lot of extra work and it will be more difficult when a user has millions of followers. So distribution of follower per user becomes a a key load parameter for discussing scalability. 

The final solution is a hybrid of both approaches. Most users’ tweets continue to be fanned out to home timelines at the time when they are posted. While tweets from any celebrities that a user may follow are fetched separately and merged with that user’s home timeline when it is read.

### How to describe performance

Once you have described the load on your system, you can investigate what happens when the load increases in two ways:
- When you increase a load parameter and keep the system resources (CPU, memory, network bandwidth, etc.) unchanged, how is the performance of your system affected?
- When you increase a load parameter, how much do you need to increase the resources if you want to keep performance unchanged?

We therefore need to think of response time not as a single number, but as a distribution of values that you can measure. So usually it is better to use *percentiles* compared to *average* because it tells you how many users experience what situations. In order to figure out how bad your outliers are, you can look at higher percentiles: the 95th, 99th, and 99.9th percentiles are common (abbreviated p95, p99, and p999).

High percentiles of response times, also known as tail latencies, are important because they directly affect users’ experience of the service - commonly the most valuable customers experience slowest requests because they have the most data. However, 99.99th percentile may be too expensive to optimize because they are easily affected by random events outside of your control.

Use scenario of percentile: in service level objectives (SLOs) and service level agreements (SLAs), contracts that define the expected performance and availability of a service which set expectations for clients of the service and allow customers to demand a refund if the SLA is not met.

Queueing delays often account for a large part of the response time at high percentiles. Due to this effect, it is important to measure response times on the client side. During testing, the load generating client needs to keep sending requests independently of the response time to be more close to real case.

### How to cope with load

Scaling up: vertical scaling, moving to a more powerful machine

Scaling out: horizontal scaling, distributing the load across multiple smaller machines

Elastic system: means that they can automatically add computing resources when they detect a load increase, whereas other systems are scaled manually (a human analyzes the capacity and decides to add more machines to the system)

The architecture of systems that operate at large scale is usually highly specific to the application—there is no such thing as a generic, one-size-fits-all scalable architecture. 

An architecture that scales well for a particular application is built around assumptions of which operations will be common and which will be rare—the load parameters. In an early-stage startup or an unproven product it’s usually more important to be able to iterate quickly on product features than it is to scale to some hypothetical future load. 

Even though scalable architecture are specific to a particular application, scalable architectures are nevertheless usually built from general-purpose building blocks, arranged in familiar patterns.

## Maintainability

Maintainability has many facets, but in essence it’s about making life better for the engineering and operations teams who need to work with the system.

To design a software that minimize maintenance cost, we will pay particular attention to three design principles:
- Operability: Make it easy for operations teams to keep the system running smoothly.
- Simplicity: Make it easy for new engineers to understand the system.
- Evolvability (also known as extensibility, modifiability, or plasticity): Make it easy for engineers to make changes to the system in the future. 

### Operability: making life easy for operations

Good operability means making routine tasks easy, allowing the operations team to focus their efforts on high-value activities. Data systems can do various things to make routine tasks easy, including:
- Providing visibility into the runtime behavior and internals of the system, with good monitoring
- Providing good support for automation and integration with standard tools
- Avoiding dependency on individual machines (allowing machines to be taken down for maintenance while the system as a whole continues running uninterrupted)
- Providing good documentation and an easy-to-understand operational model (“If I do X, Y will happen”)
- Providing good default behavior, but also giving administrators the freedom to override defaults when needed
- Self-healing where appropriate, but also giving administrators manual control over the system state when needed
- Exhibiting predictable behavior, minimizing surprises

### Simplicity: managing complexity

Complexity makes maintenance hard. Also, when the system is harder for developers to understand and reason about, there is a greater risk of introducing bugs when making a change.

Complexity is accidental if it is not inherent in the problem that the software solves (as seen by the users) but arises only from the implementation.

One of the best tools we have for removing accidental complexity is abstraction. A good abstraction can hide a great deal of implementation detail behind a clean, simple-to-understand façade. A good abstraction can also be used for a wide range of different applications. 

Not only is this reuse more efficient than reimplementing a similar thing multiple times, but it also leads to higher-quality software, as quality improvements in the abstracted component benefit all applications that use it.

### Evolvability: making change easy

The ease with which you can modify a data system, and adapt it to changing requirements, is closely linked to its simplicity and its abstractions: simple and easy-to understand systems are usually easier to modify than complex ones.

# Chapter 2 Data Models and Query Languages

Most applications are built by layering one data model on top of another. Each layer hides the complexity of the layers below it by providing a clean data model. There are many different kinds of data models, and every data model embodies assumptions about how it is going to be used.

## Relational model versus document model

The best-known data model today is SQL, based on the relational model proposed: data is organized into relations (called tables in SQL), where each relation is an unordered collection of tuples (rows in SQL).

The roots of relational databases lie in business data processing whose use cases are transaction processing (entering sales or banking transactions, airline reservations, stock-keeping in warehouses) and batch processing (customer invoicing, payroll, reporting).

Over the years, there have been many competing approaches to data storage and querying. In the 1970s and early 1980s, the network model and the hierarchical model were the main alternatives, but the relational model came to dominate them. Object databases came and went again in the late 1980s and early 1990s. XML databases appeared in the early 2000s, but have only seen niche adoption.

### NoSQL
In the 2010s, NoSQL is the latest attempt to overthrow the relational model’s dominance. A number of interesting database systems are now associated with the #NoSQL hashtag, and it has been retroactively reinterpreted as Not Only SQL.

Driving force of NoSQL:
- A need for **greater scalability** than relational databases can easily achieve, including very large datasets or very high write throughput
- A widespread preference for **free and open source** software over commercial database products
- **Specialized query** operations that are not well supported by the relational model
- Frustration with the restrictiveness of relational schemas, and a desire for a more **dynamic and expressive** data model

Impedance mismatch: if data is stored in relational tables, an awkward translation layer is required between the objects in the application code and the database model of tables, rows, and columns. Some developers feel that the JSON model reduces the impedance mismatch between the application code and the storage layer.

 The JSON representation has better *locality* than the multi-table schema because the JSON representation, all the relevant information is in one place, and one query is sufficient. The one-to-many relationships imply a tree structure in the data, and the JSON representation makes this tree structure explicit. 

### Many-to-one and many-to-many relationships

Benefit of standardize database fields:
- Consistent style and spelling
- Avoiding ambiguity (e.g., if there are several cities with the same name)
- Ease of updating—the name is stored in only one place
- Localization support because data is stored in one place
- Better search 

Anything that is meaningful to humans may need to change sometime in the future—and if that information is duplicated, all the redundant copies need to be updated. That incurs write overheads, and risks inconsistencies. Removing such duplication is the key idea behind normalization in databases.

Unfortunately, normalizing this data requires many-to-one relationships which don’t fit nicely into the document model. If the database itself does not support joins, you have to emulate a join in application code by making multiple queries to the database.

### Network model

The network model was standardized by a committee called the Conference on Data Systems Languages (CODASYL). It is also known as the CODASYL model.

In the network model, a record could have multiple parents. The links between records in the network model were not foreign keys, but more like pointers in a programming language. The only way of accessing a record was to follow a path from a root record along these chains of links. This was called an access path.

A query in CODASYL was performed by moving a cursor through the database by iterating over lists of records and following access paths. If a record had multiple parents (i.e., multiple incoming pointers from other records), the application code had to keep track of all the various relationships. This problem was that they made the code for querying and updating the database complicated and inflexible.

While in the relational model, the query optimizer automatically decides which parts of the query to execute in which order, and which indexes to use, so the application developers don't need to think about them. A key insight of the relational model was this: you only need to build a query optimizer once, and then all applications that use the database can benefit from it.

### Relational versus document databases today

#### Simpler application code

The document model is good to use when the application has a document-like structure. But it has limitations: 1) cannot refer directly to a nested item within a document; 2) poor support for join, and application level join is usually slower than a join performed by specialized code inside the database.

#### Schema flexibility

Most document databases, and the JSON support in relational databases, do not enforce any schema. No schema means that arbitrary keys and values can be added to a document, and when reading, clients have no guarantees as to what fields the documents may contain. Document database are sometimes called schemaless because of this property, but as the code that reads the data usually assumes some kind of structure—i.e., there is an implicit schema, but it is not enforced by the database (called schema-on-read). 

Schema-on-read is similar to dynamic (runtime) type checking in programming languages, whereas schema-on-write is similar to static (compile-time) type checking.

The difference between the approaches is particularly noticeable in situations where an application wants to change the format of its data. In a document database, you would just start writing new documents with the new fields and have code in the application that handles the case when old documents are read. On the other hand, in a “statically typed” database schema, you would typically perform a migration. And schema changes have a bad reputation of being slow and requiring downtime.

The schema-on-read approach is advantageous if the items in the collection don’t all have the same structure for some reason (i.e., the data is heterogeneous).

But in cases where all records are expected to have the same structure, schemas are a useful mechanism for documenting and enforcing that structure.

#### Data locality

If your application often needs to access the entire document, there is a performance advantage to this storage locality, compared to data that is split across multiple tables, multiple index lookups are required to retrieve it all, which may require more disk seeks and take more time.

The locality advantage only applies if you need large parts of the document at the same time. Because The database typically needs to load the entire document, even if you access only a small portion of it.

#### Convergence

Most relational database systems (other than MySQL) have supported XML which allows applications to use data models very similar to what they would do when using a document database.

On the document database side, RethinkDB supports relational-like joins in its query language, and some MongoDB drivers automatically resolve database references

## Query language

SQL is a declarative query language, whereas IMS and CODASYL queried the database using imperative code. 

An imperative language tells the computer to perform certain operations in a certain order. 

In a declarative query language, you just specify the pattern of the data you want—what conditions the results must meet, and how you want the data to be transformed (e.g., sorted, grouped, and aggregated)—but not how to achieve that goal. It hides implementation details of the database engine, which makes it possible for the database system to introduce performance improvements without requiring any changes to queries.

When we take parallel into consideration, imperative code is very hard to parallelize across multiple cores and multiple machines, , because it specifies instructions that must be performed in a particular order. Declarative languages have a better chance of getting faster in parallel because the database is free to use a parallel implementation of the query language.

### Declarative queries on the web

In a web browser, using declarative CSS styling is much better than manipulating styles imperatively in JavaScript.

### MapReduce querying

MapReduce is a programming model for processing large amounts of data in bulk across many machines. 

```
db.observations.mapReduce(
    function map() { 
        var year = this.observationTimestamp.getFullYear(); 
        var month = this.observationTimestamp.getMonth() + 1; 
        emit(year + "-" + month, this.numAnimals); 
    }, 
    function reduce(key, values) {
        return Array.sum(values); 
    }, 
    {
        query: { family: "Sharks" },
        out: "monthlySharkReport" 
    }
);
```
- The JavaScript function map is called once for every document that matches query, with this set to the document object.
- The map function emits a key and a value.
- The key-value pairs emitted by map are grouped by key. For all key-value pairs with the same key (i.e., the same month and year), the reduce function is called once.

Being able to use JavaScript code in the middle of a query is a great feature for advanced queries.

A usability problem with MapReduce is that you have to write two carefully coordinated JavaScript functions, which is often harder than writing a single query. Moreover, a declarative query language offers more opportunities for a query optimizer to improve the performance of a query. For these reasons, MongoDB 2.2 added support for a declarative query language called the aggregation pipeline. In this language, the previous example query looks like this:

```
db.observations.aggregate([
    { $match: { family: "Sharks" } }, 
    { $group: { 
        _id: { 
            year: { $year: "$observationTimestamp" }, 
            month: { $month: "$observationTimestamp" } 
        }, 
        totalAnimals: { $sum: "$numAnimals" } 
    } }
]);
```

## Graph-like data models

The relational model can handle simple cases of many-to-many relationships, but as the connections within your data become more complex, it becomes more natural to start modeling your data as a graph.

A graph consists of two kinds of objects: vertices (also known as nodes or entities) and edges (also known as relationships or arcs). Many kinds of data can be modeled as a graph.

Graphs are not limited to homogeneous data: an equally powerful use of graphs is to provide a consistent way of storing completely different types of objects in a single datastore.

### Property graphs

In the property graph model, each vertex consists of:
- A unique identifier
- A set of outgoing edges
- A set of incoming edges
- A collection of properties (key-value pairs)

Each edge consists of:
- A unique identifier
- The vertex at which the edge starts (the tail vertex)
- The vertex at which the edge ends (the head vertex)
- A label to describe the kind of relationship between the two vertices
- A collection of properties (key-value pairs)

Some important aspects of this model that:
- There is no schema that restricts which kinds of things can or cannot be associated.
- Given any vertex, you can efficiently find both its incoming and its outgoing edges, and thus traverse the graph.
- By using different labels for different kinds of relationships, you can store several different kinds of information in a single graph, while still maintaining a clean data model.

Those features give graphs a great deal of flexibility. Graphs are good for evolvability: as you add features to your application, a graph can easily be extended to accommodate changes in your application’s data structures.

### The Cypher query language

Cypher is a declarative query language for property graphs.

``` Cypher
CREATE
(NAmerica:Location {name:'North America', type:'continent'}), 
(USA:Location {name:'United States', type:'country' }), 
(Idaho:Location {name:'Idaho', type:'state' }), 
(Lucy:Person {name:'Lucy' }), 
(Idaho) -[:WITHIN]-> (USA) -[:WITHIN]-> (NAmerica), 
(Lucy) -[:BORN_IN]-> (Idaho)

MATCH 
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'}), 
(person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'}) 
RETURN person.name
```

As is typical for a declarative query language, you don’t need to specify such execution details when writing the query: the query optimizer automatically chooses the strategy that is predicted to be the most efficient.

### Graph queries in SQL

In a graph query, you may need to traverse a variable number of edges before you find the vertex you’re looking for - that is, the number of joins is not fixed in advance.

Since SQL:1999, this idea of variable-length traversal paths in a query can be expressed using something called recursive common table expressions. However, the syntax is very clumsy in comparison to Cypher.

### Triple-Stores and SPARQL

In a triple-store, all information is stored in the form of very simple three-part statements: (subject, predicate, object). The object is one of two things:
1. A value in a primitive datatype。In this case, he predicate and object of the triple are equivalent to the key and value of a property on the subject vertex. 
2. Another vertex in the graph. In that case, the predicate is an edge in the graph, the subject is the tail vertex, and the object is the head vertex. 

```Turtle
@prefix : <urn:example:>.
_:lucy a :Person.
_:lucy :name "Lucy".
_:lucy :bornIn _:idaho.
_:idaho a :Location.
```

The Resource Description Framework (RDF) was intended as a mechanism for different websites to publish data in a consistent format, allowing data from different websites to be automatically combined into a web of data—a kind of internet-wide “database of everything.”

SPARQL is a query language for triple-stores using the RDF data model

```SPARQL
PREFIX : <urn:example:> 

SELECT ?personName WHERE {
?person :name ?personName.
?person :bornIn / :within* / :name "United States".
?person :livesIn / :within* / :name "Europe". 
}
```

### Graph databases compared to the network model

| Aspect | CODASYL | Graph Database |
| --- | --- | --- |
| schema | a database had a schema that specified which record type could be nested within which other record type | no such restriction |
| reach a particular record | traverse one of the access paths to it | refer directly to any vertex by its unique ID, or you can use an index to find vertices with a particular value |
| order of record | the children of a record were an ordered set, so the database had to maintain that ordering | vertices and edges are not ordered | 
| query | all queries were imperative | you can write your traversal in imperative code if you want to, but most graph databases also support high-level, declarative query languages |

### The foundation: Datalog

Datalog is a much older language than SPARQL or Cypher, having been studied extensively by academics in the 1980s. It is less well known among software engineers, but it is nevertheless important, because it provides the foundation that later query languages build upon. Datalog is a subset of Prolog.

# Chapter　3 Storage and Retrieval

## Data Structures That Power Your Database

Many databases internally use a log, which is an append-only data file. It's great at writing but not good at reading.

In order to efficiently find the value for a particular key in the database, we need a different data structure: an index. The general idea behind them is to keep some additional metadata on the side, which acts as a signpost and helps you to locate the data you want. If you want to search the same data in several different ways, you may need several different indexes on different parts of the data.

An index is an additional structure that is derived from the primary data. This doesn’t affect the contents of the database; it only affects the performance of queries. Any kind of index usually slows down writes, because the index also needs to be updated every time data is written.

### Hash Indexes

The simplest possible indexing strategy is this: keep an in-memory hash map where every key is mapped to a byte offset in the data file—the location at which the value can be found.

This kind of storage engine is well suited to situations where there are a lot of writes, but there are not too many distinct keys, and the value for each key is updated frequently.

In order to avoid running out of space, we break the log into segments of a certain size by closing a segment file when it reaches a certain size, then perform compaction (throwing away duplicate keys in the log, and keeping only the most recent update for each key), and merge several segments together. 

While the merging and compaction is going on, we can still continue to serve read and write requests as normal, using the old segment files.

#### File format

It’s faster and simpler to use a binary format that first encodes the length of a string in bytes, followed by the raw string (without need for escaping).

#### Deleting records

If you want to delete a key and its associated value, you have to append a special deletion record to the data file (sometimes called a tombstone).

#### Crash recovery

If the database is restarted, the in-memory hash maps are lost. Then you need to restore the hash map by processing the entire segment file which might take a long time if the segment files are large and make server restarts painful.

Bitcask speeds up recovery by storing a snapshot of each segment’s hash map on disk, which can be loaded into memory more quickly.

#### Partially written records

The database may crash at any time, including halfway through appending a record to the log. Bitcask files include checksums, allowing such corrupted parts of the log to be detected and ignored.

#### Concurrency control

As writes are appended to the log in a strictly sequential order, a common implementation choice is to have only one writer thread and multiple read threads.

Advantage of log-structured storage engines:
- Appending and segment merging are sequential write operations, which are generally much faster than random writes, especially on magnetic spinning-disk hard drives.
- Concurrency and crash recovery are much simpler if segment files are append-only or immutable.
- Merging old segments avoids the problem of data files getting fragmented over time.

Limitation of hash index:
- The hash table must fit in memory
- Range queries are not efficient, have to look up each key individually in the hash maps.

### SSTables and LSM-Trees

Sorted String Table (SSTable): the sequence of key-value pairs is sorted by key.

Advantage of SSTable: 
- Merging segments is simple and efficient, even if the files are bigger than the available memory. The approach is like the one used in the *mergesort* algorithm.
- In order to find a particular key in the file, you no longer need to keep an index of all the keys in memory, but keep some of them and locate the range containing that key. 
- Since read requests need to scan over several key-value pairs in the requested range anyway, it is possible to group those records into a block and compress it before writing it to disk.

#### Constructing and maintaining SSTables

Maintaining a sorted structure on disk is possible, but maintaining it in memory is much easier. There are plenty of well-known tree data structures that you can use, such as red-black trees or AVL trees. With these data structures, you can insert keys in any order and read them back in sorted order.

1. When a write comes in, add it to an in-memory balanced tree data structure (memtable)
2. When the memtable gets bigger than some threshold, write it out to disk as the most recent SSTable segment file, and create a new memtable in memory.
3. For read request, firstly try the memtable, then the most recent segment, etc.
4. Run a merging and compaction process in the background to combine segment files and to discard overwritten or deleted values.

This scheme works very well. It only suffers from one problem: if the database crashes, the most recent writes (which are in the memtable but not yet written out to disk) are lost. In order to avoid that problem, we can keep a separate log on disk to which every write is immediately appended. 

Storage engines that are based on this principle of merging and compacting sorted files are often called LSM storage engines.

#### Performance optimizations

The LSM-tree algorithm can be slow when looking up keys that do not exist in the database. In order to optimize this kind of access, storage engines often use additional Bloom filters for approximating the contents of a set. It can tell you if a key does not appear in the database, and thus saves many unnecessary disk reads for nonexistent keys.

There are also different strategies to determine the order and timing of how SSTables are compacted and merged. The most common options are size-tiered and leveled compaction. In size-tiered compaction, newer and smaller SSTables are successively merged into older and larger SSTables. In leveled compaction, the key range is split up into smaller SSTables and older data is moved into separate “levels,” which allows the compaction to proceed more incrementally and use less disk space.

Log-Structured Merge-Tree (LSM-Tree) is simple and effective. It works well even when the dataset is bigger than the available memory. You can efficiently perform range queries because of the sorting property. And because the disk writes are sequential the LSM-tree can support remarkably high write throughput.

### B-Trees

The most widely used indexing structure is the B-tree.

B-trees break the database down into fixed-size blocks or pages, traditionally 4 KB in size (sometimes bigger), and read or write one page at a time. This design corresponds more closely to the underlying hardware, as disks are also arranged in fixed-size blocks. 

Each page can be identified using an address or location and it can be used to construct a tree of pages. A page contains several keys and references to child pages. Each child is responsible for a continuous range of keys, and the keys between the references indicate where the boundaries between those ranges lie.

The number of references to child pages in one page of the B-tree is called the branching factor.

If you want to update the value for an existing key in a B-tree, you search for the leaf page containing that key, change the value in that page, and write the page back to disk. If you want to add a new key, you need to find the page whose range encompasses the new key and add it to that page. If there isn’t enough free space in the page to accommodate the new key, it is split into two half-full pages, and the parent page is updated to account for the new subdivision of key ranges.

This algorithm ensures that the tree remains balanced: a B-tree with n keys always has a depth of O(log n).

#### Making B-trees reliable

The basic underlying write operation of a B-tree is to overwrite a page on disk with new data. Moreover, some operations require several different pages to be overwritten. This is a dangerous operation, because if the database crashes after only some of the pages have been written, you end up with a corrupted index.

In order to make the database resilient to crashes, it is common for B-tree implementations to include an additional data structure on disk: a write-ahead log (WAL, also known as a redo log). This is an append-only file to which every B-tree modification must be written before it can be applied to the pages of the tree itself. When the database comes back up after a crash, this log is used to restore the B-tree back to a consistent state.

An additional complication of updating pages in place is that careful concurrency control is required if multiple threads are going to access the B-tree at the same time. This is typically done by protecting the tree’s data structures with latches (lightweight locks).

#### B-tree optimizations

Instead of overwriting pages and maintaining a WAL for crash recovery, some databases use a copy-on-write scheme. A modified page is written to a different location, and a new version of the parent pages in the tree is created, pointing at the new location.

We can save space in pages by not storing the entire key, but abbreviating it. Especially when we only need to provide enough information to act as boundaries between key ranges. This allows the tree to have a higher branching factor, and thus fewer levels.

Many B-tree implementations try to lay out the tree so that leaf pages appear in sequential order on disk to boost range query. However, it’s difficult to maintain that order as the tree grows.

Additional pointers have been added to the tree. For example, each leaf page may have references to its sibling pages to the left and right, which allows scanning keys in order without jumping back to parent pages.

### Advantages of LSM-trees

Write amplification: one write to the database resulting in multiple writes to the disk over the course of the database’s lifetime.

LSM-trees are typically able to sustain higher write throughput than B-trees, partly because they sometimes have lower write amplification, and partly because they sequentially write compact SSTable files rather than having to overwrite several pages in the tree.

LSM-trees can be compressed better, and thus often produce smaller files on disk than B-trees. B-tree storage engines leave some disk space unused due to fragmentation.

### Downsides of LSM-trees

The compaction process can sometimes interfere with the performance of ongoing reads and writes. At higher percentiles the response time of queries to log-structured storage engines can sometimes be quite high, and B-trees can be more predictable.

The bigger the database gets, the more disk bandwidth is required for compaction. And it can happen that compaction cannot keep up with the rate of incoming writes.

An advantage of B-trees is that each key exists in exactly one place in the index. This aspect makes B-trees attractive in databases that want to offer strong transactional semantics: in many relational databases, transaction isolation is implemented using locks on ranges of keys, and in a B-tree index, those locks can be directly attached to the tree.

### Other Indexing Structures

Secondary indexes are often crucial for performing joins efficiently. The main difference is that keys are not unique. This can be solved in two ways: either by making each value in the index a list of matching row identifiers or by making each key unique by appending a row identifier to it.

The value in index can be one of two things: the actual row (document, vertex) in question, or a reference. In the latter case, the place where rows are stored is known as a heap file. It avoids duplicating data: each index just references a location in the heap file, and the actual data is kept in one place.

When updating a larger value without changing the key, it probably needs to be moved to a new location in the heap with enough space. In that case, either all indexes need to be updated to point at the new heap location of the record, or a forwarding pointer is left behind in the old heap location.

While in some situation, it is better to store the indexed row directly within an index which is known as clustered index. Clustered and covering indexes can speed up reads, but they require additional storage and can add overhead on writes.

The most common type of multi-column index is called a concatenated index, which simply combines several fields into one key by appending one column to another.

Fuzzy querying is search for similar keys, such as misspelled words. For example, full-text search engines commonly allow a search for one word to be expanded to include synonyms of the word, to ignore grammatical variations of words, and to search for occurrences of words near each other in the same document. Lucene is able to search text for words within a certain edit distance.

Some in-memory key-value stores are intended for caching use only, where it’s acceptable for data to be lost if a machine is restarted. Others that aim for durability can be achieved with special hardware, by writing a log of changes to disk, by writing periodic snapshots to disk, or by replicating the in-memory state to other machines.

When an in-memory database is restarted, it needs to reload its state, either from disk or over the network from a replica. Despite writing to disk, it’s still an in-memory database, because the disk is merely used as an append-only log for durability, and reads are served entirely from memory. Writing to disk also has operational advantages: files on disk can easily be backed up, inspected, and analyzed by external utilities.

In-memory database can offer big performance improvements by removing all the overheads associated with managing on-disk data structures - avoid the overheads of encoding in-memory data structures in a form that can be written to disk. Another advantage is providing data models that are difficult to implement with disk-based indexes (e.g. priority queues).

## Database for Data Analytics

OPLTP: online transaction processing; OLAP: online analytic processing

| Property	| Transaction processing systems (OLTP)	| Analytic systems (OLAP) |
| -- | -- | -- |
| Main read pattern	| Small number of records per query, fetched by key	| Aggregate over large number of records |
| Main write pattern	| Random-access, low-latency writes from user input	| Bulk import (ETL) or event stream |
| Primarily used by	| End user/customer, via web application	| Internal analyst, for decision support |
| What data represents	| Latest state of data (current point in time)	| History of events that happened over time |
| Dataset size	| Gigabytes to terabytes	| Terabytes to petabytes |

There was a trend for companies to stop using their OLTP systems for analytics purposes, and to run the analytics on a separate database instead. This separate database was called a data warehouse.

### Data Warehousing

A data warehouse is a separate database that analysts can query to their hearts’ content, without affecting OLTP operations (expensive analytics queries can harm the performance of concurrently executing transactions).

The data warehouse contains a read-only copy of the data in all the various OLTP systems in the company. Data is extracted from OLTP databases, transformed into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse. This process of getting data into the warehouse is known as Extract–Transform–Load (ETL)

A big advantage of using a separate data warehouse, rather than querying OLTP systems directly for analytics, is that the data warehouse can be optimized for analytic access patterns.

### Stars and Snowflakes: Schemas for Analytics

star schema (also known as dimensional modeling): At the center of the schema is a so-called fact table. Each row of the fact table represents an event that occurred at a particular time. Some of the columns in the fact table are attributes. Other columns in the fact table are foreign key references to other tables, called dimension tables. As each row in the fact table represents an event, the dimensions represent the who, what, where, when, how, and why of the event.

It's called “star schema” because the fact table is in the middle, surrounded by its dimension tables. A variation of this template is known as the snowflake schema, where dimensions are further broken down into subdimensions. Snowflake schemas are more normalized than star schemas, but star schemas are often preferred because they are simpler for analysts to work with.

## Column-Oriented Storage

In most OLTP databases, storage is laid out in a row-oriented fashion: all the values from one row of a table are stored next to each other. For OLAP system, usually a fact table has hundreds of columns and only query few of them at one time. So we have another idea of column-oriented storage, where values from each column are stored together, and each column file contains the rows in the same order.

### Column Compression

Besides only loading the required columns, we can further reduce the demands on disk throughput by compressing data.

Bitmap encoding: Often, the number of distinct values in a column is small compared to the number of rows. We can now take a column with *n* distinct values and turn it into *n* separate bitmaps: one bitmap for each distinct value, with one bit for each row. The bit is 1 if the row has that value, and 0 if not. 

And bitmap is very well suited for queries with logical operators. (e.g. "where ... and ..." could be fetched by calculating the bitwise AND.) This works because the columns contain the rows in the same order, so the kth bit in one column’s bitmap corresponds to the same row as the kth bit in another column’s bitmap.

If distinct value *n* is bigger, there will be a lot of zeros in most of the bitmaps (we say that they are sparse). In that case, the bitmaps can additionally be run-length encoded. (e.g. "5,4,3,3" - 5 zeros, 4 ones, 3 zeros, 3 ones, rest zeros)

### Memory bandwidth and vectorized processing

For OLAP, bottlenecks are the bandwidth for getting data from disk into memory, as well as using the bandwidth from main memory into the CPU cache, avoiding branch mispredictions and bubbles in the CPU instruction processing pipeline, and making use of single-instruction-multi-data (SIMD) instructions in modern CPUs.

Column compression allows more rows from a column to fit in the same amount of L1 cache. Operators, such as the bitwise AND and OR described previously, can be designed to operate on such chunks of compressed column data directly. This technique is known as vectorized processing.

### Sort Order in Column Storage

The data needs to be sorted an entire row at a time and the database administrator can choose the columns by which the table should be sorted, using their knowledge of common queries. 

Another advantage of sorted order is that it can help with compression of columns. If the primary sort column does not have many distinct values, then after sorting, it will have long sequences where the same value is repeated many times in a row. A simple run-length encoding could compress that column down to a few kilobytes—even if the table has billions of rows.

That compression effect is strongest on the first sort key. The second and third sort keys will be more jumbled up, and thus not have such long runs of repeated values.

Different queries benefit from different sort orders, so why not store the same data sorted in several different ways? Data needs to be replicated to multiple machines anyway, so that you don’t lose data if one machine fails. You might as well store that redundant data sorted in different ways so that when you’re processing a query, you can use the version that best fits the query pattern.

### Writing to Column-Oriented Storage

Column-oriented storage have the downside of making writes more difficult. As rows are identified by their position within a column, the insertion has to update all columns consistently.

A good solution from LSM-trees: All writes first go to an in-memory store, where they are added to a sorted structure and prepared for writing to disk. When enough writes have accumulated, they are merged with the column files on disk and written to new files in bulk. This is essentially what Vertica does.

Queries need to examine both the column data on disk and the recent writes in memory, and combine the two.

### Aggregation: Data Cubes and Materialized Views

Data warehouse queries often involve an aggregate function, such as COUNT, SUM, AVG, MIN, or MAX in SQL. It would be be better to cache the aggregation result instead of calculating them every time. 

One way of creating such a cache is a materialized view. A materialized view is an actual copy of the query results, written to disk (while virtual view of relational data model is just a shortcut for writing queries). When the underlying data changes, a materialized view needs to be updated, because it is a denormalized copy of the data. 

A common special case of a materialized view is known as a data cube or OLAP cube. Each cell contains the aggregate (e.g., SUM) of an attribute (e.g., net_price) of all facts with that fields combination (e.g. date-product-store-promotion). 

The advantage of a materialized data cube is that certain queries become very fast because they have effectively been precomputed. The disadvantage is that a data cube doesn’t have the same flexibility as querying the raw data, and use aggregates such as data cubes only as a performance boost for certain queries.

# Chapter 4 Encoding and Evolution

In a large application, code changes often cannot happen instantaneously because:
- With server-side applications, we may want to do *rolling upgrade* (aka. staged rollout) - deploy the new version to a few nodes at a time, check if it works well, gradually reach all nodes. This makes upgrade without service downtime.
- With client-side applications, users may not update

So it's possible to have ol and new version of code and data formats coexist in the system at the same time. And this is why we need to maintain compatibility in both directions:
- Backward compatibility: Newer code can read data that was written by older code. 
- Forward compatibility: Older code can read data that was written by newer code.

## Formats for Encoding Data

Programs usually work with data in (at least) two different representations:
1. In memory, data is kept in data structures like objects, lists, arrays, hash tables, trees, and so on.
2. When communicate through network, data is sent as some kind of self-contained sequence of bytes (for example, a JSON document).

The translation from the in-memory representation to a byte sequence is called encoding (also known as serialization or marshalling), and the reverse is called decoding (parsing, deserialization, unmarshalling).

### Language-Specific Formats

Many programming languages come with built-in support for encoding in-memory objects into byte sequences. It's easy to use but has disadvantages:
- Since the encoding is tied to a particular language, it will be difficult to read te data in another language.
- The ability to restore arbitrary classes becomes a source of security problem.
- Versioning data is often an afterthought since it is intend for quick an easy use. Forward and backward compatibility may not be supported.
- Efficiency is also often an afterthought.

### JSON, XML, and Binary Variants

JSON, XML, and CSV are widely known, widely supported, textual formats, and thus somewhat human-readable. They have some subtle problems:
- There is a lot of ambiguity around the encoding of numbers. 
    - XML and CSV don't distinguish number and string. 
    - JSON doesn't distinguish integers and floating-point numbers
    - Large number can not be parsed accurately
- JSON and XML don’t support binary strings (sequences of bytes without a character encoding)
- XML and JSON have optional schema support. These schema languages are quite powerful, and thus quite complicated to learn and implement. Without them, application hardcode the apporiate encoding/decoding logic.
- CSV does not have any schema, so it is up to the application to define the meaning of each row and column.

Even though JSON, XML, and CSV have some problems, it’s likely that they will remain popular. Because the difficulty of getting different organizations to agree on anything outweighs most other concerns.

### Binary encoding - Thrift and Protocol Buffers

For data that is used only internally, you might consider a lowest-common-denominator encoding format to get benefit like compactness or fast parsing.

There are some binary encodings for JSON. The basic idea is to have some byte indicating the data type and length. Since they don’t prescribe a schema, they need to include all the object field names within the encoded data which make the space reduction slight. Given it lose human-readability, it's not worth to do so.

Apache Thrift and Protocol Buffers (protobuf) are binary encoding libraries that require a schema. Thrift and Protocol Buffers each come with a code generation tool that takes a schema definition like the ones shown here, and produces classes that implement the schema in various programming languages. Your application code can call this generated code to encode or decode records of the schema.

These two encoding contain only *field tag* (numbers) instead of field name which save space. One detail to note is that require/optional fields are encoded in the same way, checking is performed ar runtime.

How Thrift and Protocol Buffers handle schema evolution:
- Forward compatibility: when new field is added, old code just ignore new field so that old code can read records that were written by new code.
- Backward compatibility: as long as each field has a unique tag number, new code can always read old data, because the tag numbers still have the same meaning. The only detail is that if you add a new field, you should make it optional or have a default value.
- Remove a field: you can only remove a field that is optional, and you can never use the same tag number again.
- Changing the datatype of a field: it's possible but values may lose precision or get truncated.

Protocol Buffers is that it does not have a list or array datatype, but instead has a repeated marker for fields which has a nice effect to change an optional field into a repeated field. Thrift has a dedicated list datatype which doesn't allow the evolution as Protocol Buffers does, but it has the advantage of supporting nested lists.

### Binary encoding - Avro

Apache Avro is another binary encoding format that is interestingly different from Protocol Buffers and Thrift. 

There is nothing to identify fields (no tag numbers) or their datatypes. To parse the binary data, you go through the fields in the order that they appear in the schema and use the schema to tell you the datatype of each field. This means that the binary data can only be decoded correctly if the code reading the data is using the exact same schema as the code that wrote the data.

Writer’s schema is used to encode the data and reader’s schema is used to decode the data. The key idea with Avro is that the writer’s schema and the reader’s schema don’t have to be the same—they only need to be compatible.

When data is decoded (read), the Avro library resolves the differences by looking at the writer’s schema and the reader’s schema side by side and translating the data from the writer’s schema into the reader’s schema (e.g. match field by field name, ignore field that only appear in writer's schema, fill field that writer's schema doesn't contain with default value).

Schema evolution in Avro:
- Forward compatibility: have a new version of the schema as writer and an old version of the schema as reader.
- Backward compatibility: have a new version of the schema as reader and an old version as writer.
- To maintain compatibility, you may only add or remove a field that has a default value.

In some programming languages, null is an acceptable default for any variable, but this is not the case in Avro: if you want to allow a field to be null, you have to use a union type (e.g. `union { null, long, string } field;`). You can only use null as a default value if it is one of the branches of the union. This is a little verbose but it helps prevent bugs by being explicit about what can and cannot be null.

How the reader know the writer's schema depends on the context:
- Large file with lots of records: the writer of that file can just include the writer’s schema once at the beginning of the file since all records encoded with the same schema.
- Database with individually written records: include a version number at the beginning of every encoded record, and to keep a list of schema versions in your database.
- Sending records over a network connection: negotiate the schema version on connection setup.

Avro is friendlier to dynamically generated schemas. Since the fields are identified by name, it's easy to generate a new Avro schema based on database schema change, and the updated writer’s schema can still be matched up with the old reader’s schema.

Thrift and Protocol Buffers rely on code generation to implement schema in particular language. This is useful in statically typed languages because it allows efficient in-memory structures to be used for decoded data, and it allows type checking and autocompletion in IDEs.

Avro provides optional code generation for statically typed programming languages, but it can be used just as well without any code generation because object container file (embed with its writer's schema) self-describing since it includes all the necessary metadata.

### The Merits of Schemas

Nice property of binary encodings based on schemas:
- They can be much more compact since omitting field name.
- The schema is a valuable form of documentation.
- Keeping a database of schemas allows you to check forward and backward compatibility of schema changes before deployment.
- The ability to generate code from the schema enables type checking for statically typed programming language.

## Modes of Dataflow

### Dataflow Through Databases

In a database, the process that writes to the database encodes the data, and the process that reads from the database decodes it. Backward compatibility is clearly necessary here; otherwise your future self won’t be able to decode what you previously wrote.

It's possible to entirely update the application code, but very old data may still be in the database in the original encoding since it's expansive to rewrite data into a new schema. This observation is sometimes summed up as data outlives code.

Schema evolution thus allows the entire database to appear as if it was encoded with a single schema.

### Dataflow Through Services: REST and RPC

The most common arrangement in network communication is to have two roles: clients and servers. The API (Application Programming Interface) exposed by the server is known as a service.

Moreover, a server can itself be a client to another service which is often used to decompose a large application into smaller services by area of functionality. This way of building applications has traditionally been called a service-oriented architecture (SOA), more recently refined and rebranded as microservices architecture. 

A key design goal of a service-oriented/microservices architecture is to make the application easier to change and maintain by making services independently deployable and evolvable. In other words, we should expect old and new versions of servers and clients to be running at the same time.

#### Web services

When HTTP is used as the underlying protocol for talking to the service, it is called a web service. There are two popular approaches to web services: REST and SOAP.

REST is not a protocol, but rather a design philosophy that emphasizes simple data formats, using URLs for identifying resources and using HTTP features for cache control, authentication, and content type negotiation. An API designed according to the principles of REST is called RESTful.

By contrast, SOAP is an XML-based protocol for making network API requests which aims to be independent from HTTP and avoids using most HTTP features. The API of a SOAP web service is described using an XML-based language called the Web Services Description Language, or WSDL. As WSDL is not designed to be human-readable, and as SOAP messages are often too complex to construct manually, users of SOAP rely heavily on tool support, code generation, and IDEs.

Remote procedure call (RPC) tries to make a request to a remote network service look the same as calling a function or method in your programming language, within the same process (this abstraction is called location transparency). But a network request is very different from a local function call:
- A network request is unpredictable
- A local function call either returns a result, or throws an exception, or never returns. A network request has another possible outcome: it may return without a result, due to a timeout.
- Retry a failed network request may cause problem unless you build a mechanism for deduplication (idempotence) into the protocol.
- A network request is much slower than a function call, and its latency is also wildly variable.
- Parameter passing is different. In function call, you pass references (pointers), while in network request, you pass encoded byte sequence.
- The client and the service may be implemented in different programming languages, so the RPC framework must translate datatypes from one language into another.

All of these factors mean that there’s no point trying to make a remote service look too much like a local object. Part of the appeal of REST is that it doesn’t try to hide the fact that it’s a network protocol.

#### Current directions for RPC

The new generation of RPC frameworks is more explicit about the fact that a remote request is different from a local function call. For example, Finagle and Rest.li use futures (promises) to encapsulate asynchronous actions that may fail. Futures also simplify situations where you need to make requests to multiple services in parallel.

#### Data encoding and evolution for RPC

It is reasonable to assume that all the servers will be updated first, and all the clients second. Thus, you only need backward compatibility on requests, and forward compatibility on responses. The backward and forward compatibility properties of an RPC scheme are inherited from whatever encoding it uses. 

If a compatibility-breaking change is required, the service provider often ends up maintaining multiple versions of the service API side by side. For RESTful APIs, common approaches are to use a version number in the URL or in the HTTP Accept header.

### Message-Passing Dataflow

Asynchronous message-passing systems are somewhere between RPC and databases - a client’s request (usually called a message) is delivered to another process with low latency, the message is not sent via a direct network connection, but goes via an intermediary called a message broker (also called a message queue or message-oriented middleware).

Using a message broker has several advantages compared to direct RPC:
- It can act as a buffer if the recipient is unavailable or overloaded, and thus improve system reliability.
- It can automatically redeliver messages to a process that has crashed, and thus prevent messages from being lost.
- It avoids the sender needing to know the IP address and port number of the recipient (which is particularly useful in a cloud deployment).
- It allows one message to be sent to several recipients.
- It logically decouples the sender from the recipient.

This communication pattern is asynchronous: the sender doesn’t wait for the message to be delivered, but simply sends it and then forgets about it.

#### Message brokers

In general, one process sends a message to a named queue or topic, and the broker ensures that the message is delivered to one or more consumers of or subscribers to that queue or topic. There can be many producers and many consumers on the same topic.

A topic provides only one-way dataflow. However, a consumer may itself publish messages to another topic, or to a reply queue that is consumed by the sender of the original message. Message brokers typically don’t enforce any particular data model—a message is just a sequence of bytes with some metadata, so you can use any encoding format.

#### Distributed actor frameworks

The actor model is a programming model for concurrency in a single process. problems of race conditions, locking, and deadlock), logic is encapsulated in actors. Each actor typically represents one client or entity, it may have some local state, and it communicates with other actors by sending and receiving asynchronous messages. Message delivery is not guaranteed.

Distributed actor frameworks is used to scale an application across multiple nodes. The same message-passing mechanism is used, no matter whether the sender and recipient are on the same node or different nodes. Location transparency works better in the actor model than in RPC, because the actor model already assumes that messages may be lost.

# Chapter 5 Replication

Replication: keep a copy of the same data on multiple machines that are connected via a network.

Reason of replication:
- Reduce latency by keeping data geographically close to your users
- Increase availability by providing service even if some machine fail
- Increase read throughput by scaling out the number of machines that can serve read queries

## Leaders and Followers

Replica: each node that stores a copy of the database

Leader-based replication (aka. active/passive or master–slave replication)
1. One of the replicas is designated the leader. Clients must send their write requests to the leader, which first writes the new data to its local storage.
2. The other replicas are known as followers. Leader sends the data change to all of its followers as part of a replication log or change stream. Each follower updates its local copy of the database accordingly, by applying all writes in the same order as they were processed on the leader.
3. Clients can send read requests to either the leader or any of the followers.

Finally, leader-based replication is not restricted to only databases: distributed message brokers such as Kafka and RabbitMQ highly available queues also use it.

### Synchronous Versus Asynchronous Replication

Synchronous replication: the leader waits until followers have confirmed before reporting success to the client. 

Asynchronous replication: the leader sends the message, but doesn’t wait for a response from the follower.

Advantage of synchronous replication is that followers are guaranteed to have an up-to-date copy of the data. But the write cannot be processed if one follower doesn't response, which make it impractical. So in practice, synchronous replication usually means that one of the followers is synchronous, and the others are asynchronous (semi-synchronous).

Completely asynchronous replication is not guaranteed to be durable because if the leader fail and is not recoverable, it's possible to lose data. While it has the advantage that it process write even if all followers fall behind.

### Setting Up New Followers

Simply copying data files from one node to another is typically not sufficient: clients are constantly writing to the database, and the data is always in flux. If we make the file consistent by locking the database, it would go against our goal of high availability.

Approach of setting up a follower without downtime:
1. Take a consistent snapshot of the leader’s database at some point in time.

2. Copy the snapshot to the new follower node.

3. The follower connects to the leader and requests all the data changes that have happened since the snapshot was taken based on the snapshot's position in the leader's replication log.

4. When the follower has processed the backlog of data changes since the snapshot, we say it has caught up.

### Handling Node Outages

On each follower's local disk, it keeps a log of the data changes it has received from the leader.

#### Follower failure: Catch-up recovery

Thus, the follower can connect to the leader and request all the data changes that occurred during the time when the follower was disconnected based on local log.

#### Leader failure: Failover

Failover process:
1. Determining that the leader has failed when it doesn't respond for some period of time.

2. Choosing a new leader through an election process or appointing a previously elected controller node. The best candidate for leadership is usually the replica with the most up-to-date data.

3. Reconfiguring the system to use the new leader. Clients now need to send their write requests to the new leader. If the old leader comes back, it should become a follower and recognizes the new leader.

Things that can go wrong with failover:
- If the former leader rejoins the cluster after a new leader has been chosen, what should happen to those writes? The most common solution is for the old leader’s unreplicated writes to simply be discarded, which may violate clients’ durability expectations.
- Discarding writes is especially dangerous if other storage systems outside of the database need to be coordinated with the database contents. (e.g. out-of-date follower become leader, use autoincrementing counter to assign primary keys may resulting in reusing the same key)
- It could happen that two nodes both believe that they are the leader. This situation is called *split brain*. It is dangerous: if both leaders accept writes, and there is no process for resolving conflicts, data is likely to be lost or corrupted. Common solution is to shut down one node if two leaders are detected.
- What is the right timeout before the leader is declared dead? A long one means a longer time to recovery, while a short one may cause unnecessary failovers.

There are no easy solutions to these problems. For this reason, some operations teams prefer to perform failovers manually, even if the software supports automatic failover.

### Implementation of Replication Logs

#### Statement-based replication

The leader logs every write request (statement) that it executes and sends that statement log to its followers. It's simple but has some problems:
- Any statement that calls a nondeterministic function (e.g.) NOW(), RAND()), is likely to generate a different value on each replica.
- If statements use an autoincrementing column, or if they depend on the existing data in the database (e.g., UPDATE … WHERE <some condition>), they must be executed in exactly the same order on each replica, or else they may have a different effect.
- Statements that have side effects (e.g., triggers, stored procedures, user-defined functions) may result in different side effects occurring on each replica.

#### Write-ahead log (WAL) shipping

The log is an append-only sequence of bytes containing all writes to the database. The main disadvantage is that log describes the data on a very low level: a WAL contains details of which bytes were changed in which disk blocks. This makes replication closely coupled to the storage engine. If the database changes its storage format from one version to another, it is typically not possible to run different versions of the database software on the leader and the followers.

#### Logical (row-based) log replication

A logical log for a relational database is usually a sequence of records describing writes to database tables at the granularity of a row:
- For insertion, it contains the new values of all columns.
- For deletion, it contains enough information to uniquely identify the row that was deleted.
- For update, it contains both identifying information and new value.
- For transaction, it contains rows information along with a record indicating that the transaction was committed.

The advantage is that it can be decoupled from the storage engine internals which allowing the leader and the follower to run different versions of the database software. It is also easier for external applications to parse which is useful when you want to send the contents of a database to an external system (e.g. data warehouse).

#### Trigger-based replication

A trigger lets you register custom application code that is automatically executed when a data change, like logging this change into a separate table, which could be read by an external process. This approach provides more flexibility, however, it also has greater overheads than other approaches.

## Problems with Replication Lag

Eventual consistency: —if you stop writing to the database and wait a while, the followers will eventually catch up and become consistent with the leader.

Replication lag: the delay between a write happening on the leader and being reflected on a follower.

### Reading Your Own Writes

When new data is submitted, it must be sent to the leader, but when the user views the data, it can be read from a follower which possibly don't have the user's write yet. 

In this situation, we need *read-after-write* consistency (aka. read-your-writes consistency). It reassures the user that their own input has been saved correctly. And it makes no promises about other users.

Ways to achieve read-after-write consistency:
- When reading something that the user may have modified, read it from the leader which require knowledge of whether something may be modified.
- If most things are potentially editable, we could use some criteria to decide whether to read from the leader (only read within one minute after the last update should go to the leader).
- Client remember the timestamp of last write, and the system can ensure that the replica serving any reads for that user reflects updates at least until that timestamp. The timestamp could be a logical timestamp indicating ordering of writes, or actual system clock (may be unreliable).
- If your replicas are distributed across multiple datacenters, any request that needs to be served by the leader must be routed to the datacenter that contains the leader.

Another complication arises when the same user is accessing your service from multiple devices. In this case, we need *cross-device read-after-write* consistency - if the user enters some information on one device and then views it on another device, they should see the information they just entered.

### Monotonic Reads

Monotonic reads: a guarantee that if one user makes several reads in sequence, they will not see time go backward. It’s a lesser guarantee than strong consistency, but a stronger guarantee than eventual consistency.

One possible approach to achieve so is that each user always makes their reads from the same replica. However, if that replica fails, the user’s queries will need to be rerouted to another replica.

### Consistent Prefix Reads

Consistent prefix reads: a guarantee that if a sequence of writes happens in a certain order, then anyone reading those writes will see them appear in the same order. 

However, in many distributed databases, different partitions operate independently, so there is no global ordering of writes. One solution is to make sure that any writes that are causally related to each other are written to the same partition

### Solutions for Replication Lag

We want transaction because it would be better if application developers didn’t have to worry about subtle replication issues and could just trust their databases to “do the right thing.” However, in the move to distributed (replicated and partitioned) databases, transactions are too expensive in terms of performance and availability, and asserting that eventual consistency is inevitable in a scalable system.

## Multi-Leader Replication

Leader-based replication has one major downside: there is only one leader, and all writes must go through it. A natural extension of the leader-based replication model is to allow more than one node to accept writes and each node that processes a write still forward that data change to all the other nodes..

### Use Cases for Multi-Leader Replication

#### Multi-datacenter operation

In a multi-leader configuration, you can have a leader in each datacenter. Between datacenters, each datacenter’s leader replicates its changes to the leaders in other datacenters. This approach has several benefits:
- Performance. Every write can be processed in the local datacenter, which hide network delay from users and reduce latency. 
- Tolerance of datacenter outages.
- Tolerance of network problems. 

It also has a big downside: the same data may be concurrently modified in two different datacenters, and those write conflicts must be resolved. There are often subtle configuration pitfalls and surprising interactions with other database features - autoincrementing keys, triggers, and so on.

#### Clients with offline operation

Another situation in which multi-leader replication is appropriate is if you have an application that needs to continue to work while it is disconnected from the internet. If you make any changes while you are offline, they need to be synced with a server and your other devices when the device is next online.

In this case, every device has a local database that acts as a leader.

#### Collaborative editing

Real-time collaborative editing has a lot in common with the previously mentioned offline editing use case. When user make a change, it is instantly applied to their local replica, and asynchronously replicated to the server and other users.

### Handling Write Conflicts

The biggest problem with multi-leader replication is that write conflicts can occur, which means that conflict resolution is required.

Write conflict is usually detected asynchronously - writes are succeed and conflict is detect at some later time point. Because synchronous detection means wait for conflict resolve before return success, which will lose the main advantage of multi-leader replication: allowing each replica to accept writes independently.

#### Conflict avoidance

The simplest strategy for dealing with conflicts is to avoid them: if the application can ensure that all writes for a particular record go through the same leader, then conflicts cannot occur.

However, sometimes you might want to change the designated leader for a record. In this situation, conflict avoidance breaks down, and you have to deal with the possibility of concurrent writes on different leaders.

#### Converging toward a consistent state

In a multi-leader configuration, there is no defined ordering of writes, so it’s not clear what the final value should be. If each replica simply applied writes in the order that it saw the writes, the database would end up in an inconsistent state which is unacceptable. Thus, the database must resolve the conflict in a convergent way, which means that all replicas must arrive at the same final value when all changes have been replicated.

Ways to achieve convergent conflict resolution:
- Give each write a unique ID, pick the write with the highest ID as the winner. If a timestamp is used, this technique is known as *last write wins* (LWW). Although this approach is popular, it is dangerously prone to data loss.
- Give each replica a unique ID, and let writes that originated at a higher-numbered replica always take precedence over writes that originated at a lower-numbered replica. This approach also implies data loss.
- Somehow merge the values together.
- Record the conflict in an explicit data structure that preserves all information, and write application code that resolves the conflict at some later time.

#### Custom conflict resolution logic

As the most appropriate way of resolving a conflict may depend on the application, you can write conflict resolution logic using application code. That code may be executed on write or on read:
- On write: As soon as the database system detects a conflict in the log of replicated changes, it calls the conflict handler.
- On read: All the conflicting writes are stored. The next time the data is read, these multiple versions of the data are returned to the application.

### Multi-Leader Replication Topologies

Replication topology: describes the communication paths along which writes are propagated from one node to another.

The most general topology is all-to-all, in which every leader sends its writes to every other leader. However, more restricted topologies are also used: 1) circular topology, in which each node receives writes from one node and forwards those writes (plus its own) to one other node, 2) star topology, in which one designated root node forwards writes to all of the other nodes (the star topology can be generalized to a tree).

In circular and star topologies, a write may need to pass through several nodes before it reaches all replicas. Each node is given a unique identifier, and in the replication log, each write is tagged with the identifiers of all the nodes it has passed through so that a node could ignore processed data.

A problem with circular and star topologies is that if just one node fails, it can interrupt the flow of replication messages. On the other hand, all-to-all topologies can have issues that some network links may be faster than others (e.g., due to network congestion), with the result that some replication messages may “overtake” others (will cause problem when the two messages have causality relationship). 

## Leaderless Replication

Leaderless replication: the client directly sends its writes to several replicas, or have a coordinator node does this on behalf of the client, and allow any replica to directly accept writes from clients.

### Writing to the Database When a Node Is Down

When client sends the write to all replicas in parallel, we consider the write to be successful after client received *w* ok responses, and simply ignores the fact that one of the replicas missed the write.

When client send read request, it will send to several nodes in parallel to avoid reading stale value. Version numbers are used to determine which value is newer

Two mechanisms are often used in Dynamo-style datastores to ensure that eventually all the data is copied to every replica:
- Read repair: When client notice a replica has a stale value, it will write the newer value back to that replica. This approach works well for values that are frequently read.
- Anti-entropy process: have a background process that constantly looks for differences in the data between replicas. Without an anti-entropy process, values that are rarely read may be missing from some replicas and thus have reduced durability

#### Quorums for reading and writing

More generally, if there are *n* replicas, every write must be confirmed by *w* nodes to be considered successful, and we must query at least *r* nodes for each read. As long as *w + r > n*, we expect to get an up-to-date value when reading. Reads and writes that obey these *r* and *w* values are called quorum reads and writes.

A common choice is to make *n* an odd number (typically 3 or 5) and to set* w = r = (n + 1) / 2* (rounded up). The quorum condition, *w + r > n*, allows the system to tolerate unavailable nodes.

It's possible to set set *w* and *r* to smaller numbers, so that *w + r ≤ n*, then you are more likely to read stale values because it’s more likely that your read didn’t include the node with the latest value. On the upside, this configuration allows lower latency and higher availability.

However, even with w + r > n, there are likely to be edge cases where stale values are returned:
- If a sloppy quorum is used, there is no longer a guaranteed overlap between the *r* nodes and the *w* nodes.
- If two writes occur concurrently, it is not clear which one happened first. If a winner is picked based on a timestamp (last write wins), writes can be lost due to clock skew
- If a write happens concurrently with a read, the write may be reflected on only some of the replicas. It’s undetermined whether the read returns the old or the new value.
- If a write overall succeeded on fewer than w replicas, it is not rolled back on the replicas where it succeeded.
- If a node carrying a new value fails, and its data is restored from a replica carrying an old value, the number of replicas storing the new value may fall below w, breaking the quorum condition.

Dynamo-style databases are generally optimized for use cases that can tolerate eventual consistency. The parameters w and r allow you to adjust the probability of stale values being read, but it’s wise to not take them as absolute guarantees.

From an operational perspective, it’s important to monitor whether your databases are returning up-to-date results. For leader-based replication, the database typically exposes metrics for the replication lag, which you can feed into a monitoring system. This is possible because writes are applied to the leader and to followers in the same order. However, in systems with leaderless replication, there is no fixed order in which writes are applied, which makes monitoring more difficult.

#### Sloppy Quorums and Hinted Handoff

However, quorums (as described so far) are not as fault-tolerant as they could be. A network interruption can easily cut off a client from a large number of database nodes, so the client can no longer reach a quorum.

Sloppy quorum: writes and reads still require w and r successful responses, but those may include nodes that are not among the designated n “home” nodes for a value.

Hinted handoff: once the network interruption is fixed, any writes that one node temporarily accepted on behalf of another node are sent to the appropriate “home” nodes.

Sloppy quorums are particularly useful for increasing write availability. The downside is that a read may see stale data before the hinted handoff has completed.

#### Multi-datacenter operation

Leaderless replication is also suitable for multi-datacenter operation, since it is designed to tolerate conflicting concurrent writes, network interruptions, and latency spikes. 

One way is to have the number of replicas n includes nodes in all datacenters, then read request only waits for acknowledgment from a quorum of nodes within its local datacenter, and configure write request to be asynchronously. Another way is to keeps all communication between clients and database nodes local to one datacenter, so n describes the number of replicas within one datacenter, and have cross-datacenter replication happening between database clusters asynchronously in the background.

### Detecting Concurrent Writes

#### Last write wins (discarding concurrent writes)

Last write wins (LWW): In some situation, it doesn’t really make sense to say that which write happened “first”: we say the writes are concurrent, so their order is undefined. Then, we can force an arbitrary order on them and always keep the most "recent" one.

LWW achieves the goal of eventual convergence, but at the cost of durability: if there are several concurrent writes to the same key, even if they were all reported as successful to the client, only one of the writes will survive and the others will be silently discarded.

If losing data is not acceptable, LWW is a poor choice for conflict resolution. The only safe way of using LWW is to ensure that a key is only written once and thereafter treated as immutable, thus avoiding any concurrent updates to the same key.

#### The “happens-before” relationship and concurrency

An operation A *happens before* another operation B if B knows about A, or depends on A, or builds upon A in some way. And two operations are *concurrent* if neither happens before the other.

Assume we only have one server, it could use a version number to determine whether two operations are concurrent:
- The server maintains a version number for every key, increments the version number every time that key is written.
- When a client reads a key, the server returns all values that have not been overwritten, as well as the latest version number.
- When a client writes a key, it must include the version number from the prior read, and it must merge together all values that it received in the prior read.
- When the server receives a write with a particular version number, it can overwrite all values with that version number or below, and keep all values with a higher version number.

In this case, it's up to the client to merge concurrent values. A simple approach is to just pick one of the values based on a version number or timestamp. With the example of a shopping cart, a reasonable approach to merging siblings is to just take the union. However, if you want to allow people to also remove things from their carts, you need to leave a marker with an appropriate version number to indicate that the item has been removed when merging siblings.

#### Version vectors

When we have multiple replica, we use a version number per replica as well as per key. Each replica increments its own version number when processing a write, and also keeps track of the version numbers it has seen from each of the other replicas.

Version vectors: the collection of version numbers from all the replicas. Version vectors are sent from the database replicas to clients when values are read, and need to be sent back to the database when a value is subsequently written.
