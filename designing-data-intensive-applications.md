# Designing Data-Intensive Applications

[Chapter 1 Reliable, Scalable, and Maintainable Applications](#chapter-1-reliable-scalable-and-maintainable-application)
- [Reliability](#reliability)
- [Scalability](#scalability)
- [Maintainability](#maintainability)

[Chapter 2 Data Models and Query Languages](#chapter-2-data-models-and-query-languages)
- [Relational Model Versus Document Model](#relational-model-versus-document-model)
- [Query language](#query-language)
- [Graph-like data models](#graph-like-data-models)


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

### How to describe Performance

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

### Operability: Making Life Easy for Operations

Good operability means making routine tasks easy, allowing the operations team to focus their efforts on high-value activities. Data systems can do various things to make routine tasks easy, including:
- Providing visibility into the runtime behavior and internals of the system, with good monitoring
- Providing good support for automation and integration with standard tools
- Avoiding dependency on individual machines (allowing machines to be taken down for maintenance while the system as a whole continues running uninterrupted)
- Providing good documentation and an easy-to-understand operational model (“If I do X, Y will happen”)
- Providing good default behavior, but also giving administrators the freedom to override defaults when needed
- Self-healing where appropriate, but also giving administrators manual control over the system state when needed
- Exhibiting predictable behavior, minimizing surprises

### Simplicity: Managing Complexity

Complexity makes maintenance hard. Also, when the system is harder for developers to understand and reason about, there is a greater risk of introducing bugs when making a change.

Complexity is accidental if it is not inherent in the problem that the software solves (as seen by the users) but arises only from the implementation.

One of the best tools we have for removing accidental complexity is abstraction. A good abstraction can hide a great deal of implementation detail behind a clean, simple-to-understand façade. A good abstraction can also be used for a wide range of different applications. 

Not only is this reuse more efficient than reimplementing a similar thing multiple times, but it also leads to higher-quality software, as quality improvements in the abstracted component benefit all applications that use it.

### Evolvability: Making Change Easy

The ease with which you can modify a data system, and adapt it to changing requirements, is closely linked to its simplicity and its abstractions: simple and easy-to understand systems are usually easier to modify than complex ones.

# Chapter 2 Data Models and Query Languages

Most applications are built by layering one data model on top of another. Each layer hides the complexity of the layers below it by providing a clean data model. There are many different kinds of data models, and every data model embodies assumptions about how it is going to be used.

## Relational Model Versus Document Model

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

### Many-to-One and Many-to-Many Relationships

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

### Relational Versus Document Databases Today

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

### MApReduce querying

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
