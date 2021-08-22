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

The ease with which you can modify a data system, and adapt it to changing requirements, is closely linked to its simplicity and its abstractions: simple and easy-tounderstand systems are usually easier to modify than complex ones.

# Chapter 2

Various techniques, architectures, and algorithms that are used in order to achieve reliability, scalability, and maintainability。
