
# Transaction

## Atomicity

The system can only be in the state it was before the operation or after the operation, not something in between.

## Consistency

If a transaction starts with a database that is valid according to these invariants, and any writes during the transaction preserve the validity, then you can be sure that the invariants are always satisfied.

## Isolation 
concurrently executing transactions are isolated from each other
serializable isolation: The database ensures that when the transactions have committed, the result is the same as if they had run serially

## Durability

data can be stored without fear of losing it.

once a transaction has committed successfully, any data it has written will not be forgotten, even if there is a hardware fault or the database crashes.

=====

transaction is a abstraction layer and hide lots errors from client.

## Dirty reads

One client reads another client’s writes before they have been committed.

## Dirty writes

One client overwrites data that another client has written, but not yet committed.

## Read skew (nonrepeatable reads)

A client sees different parts of the database at different points in time.

## Lost updates

Two clients concurrently perform a read-modify-write cycle. One overwrites the other’s write without incorporating its changes, so data is lost.

prevention: 1) atomic update operation. 2) explicit lock (select ... for update). 3) detect a lost update & abort that txn

## Write skew & Phantom reads

lost update can be viewed as a special case of write skew

A transaction reads objects that match some search condition. Another client makes a write that affects the results of that search.

A transaction reads something, makes a decision based on the value it saw, and writes the decision to the database. However, by the time the write is made, the premise of the decision is no longer true.

prevention: 1) constraints / triggers. 2) explicitly lock the rows that the transaction depends on (select ... for update). 3) predicate lock that belongs to all objects that match some search condition & applies even to objects that do not yet exist in the database, but which might be added in the future (phantoms), and it could be replaced by Index-range locks which match greater set of object but better performance

optimistic: detect a transaction may have acted on an outdated premise and abort the transaction in that case. 
(track from read vs write)
1. Detecting reads of a stale MVCC object version. track when a transaction ignores another transaction’s writes due to MVCC visibility rules. When the transaction wants to commit, the database checks whether any of the ignored writes have now been committed. If so, the transaction must be aborted
2. Detecting writes that affect prior reads. use the index entry record the fact that transactions 42 and 43 read this data. When a transaction writes to the database, it must look in the indexes for any other transactions that have recently read the affected data. notifies the transactions that the data they read may no longer be up to date.


========
## Read Committed

No dirty reads - 1) check if write lock exist when it want to read (but a long-running txn can block lots other txns). 2) remember old & new not-yet-committed value, return old value for read.
No writes - prevented by row-level locks that will be held until commit / abort.

problem: nonrepeatable read or read skew

## Snapshot Isolation and Repeatable Read

prevent read skew (the data on which it operates is changing at the same time as the query is executing).

each transaction reads from a consistent snapshot of the database. sees all the data that was committed in the database at the start of the transaction. 

Snapshot isolation is a boon for long-running, read-only queries such as backups and analytics.

a key principle of snapshot isolation is readers never block writers, and writers never block readers.

### implementation: 

basic idea: maintains several versions of an object side by side, this technique is known as multi-version concurrency control (MVCC).

When a transaction is started, it is given a unique, always-increasing viitransaction ID (txid). the data it writes is tagged with the transaction ID of the writer. 

transaction IDs are used to decide which objects it can see and which are invisible.

write from txn in progress (running / aborted), later is ignored.

<!-- 1. At the start of each transaction, the database makes a list of all the other transactions that are in progress (not yet committed or aborted) at that time. Any writes that those transactions have made are ignored, even if the transactions subsequently commit.

2. Any writes made by aborted transactions are ignored.

3. Any writes made by transactions with a later transaction ID (i.e., which started after the current transaction started) are ignored, regardless of whether those transactions have committed.

4. All other writes are visible to the application’s queries. -->

### how to handle index

pointer vs append-only/copy-on-write (every txn has it's own b-tree)

## Literally executing transactions in a serial order

execute only one transaction at a time, in serial order, on a single thread

reason that it become possible: 1) RAM became cheap enough. 2) OLTP transactions are usually short and only make a small number of reads and writes

to make txn quick: don't allow interactive multi-statement transactions & encapsulating transactions in stored procedures (db special language is ugly, but some db allow using common pl. require procedure to be deterministic). keep db in memory

Cross-partition transactions are possible, but there is a hard limit

## Two-phase locking

performance is not good: transaction throughput and response times of queries are significantly worse under two-phase locking than under weak isolation. (due to the overhead of acquiring and releasing all those locks, but more importantly due to reduced concurrency)

when one transaction has to wait on another, there is no limit on how long it may have to wait. databases running 2PL can have quite unstable latencies, and they can be very slow at high percentiles

## Serializable snapshot isolation (SSI)

, transactions continue anyway, in the hope that everything will turn out all right. When a transaction wants to commit, the database checks whether anything bad happened (i.e., whether isolation was violated); if so, the transaction is aborted and has to be retried. Only transactions that executed serializably are allowed to commit.

It performs badly if there is high contention. However, if there is enough spare capacity, and if contention between transactions is not too high, optimistic concurrency control techniques tend to perform better than pessimistic ones.

advantage: 
Compared to two-phase locking: one transaction doesn’t need to block waiting for locks held by another transaction. read-only queries can run on a consistent snapshot without requiring any locks, which is very appealing for read-heavy workloads.

Compared to serial execution: not limited to the throughput of a single CPU core

The rate of aborts significantly affects the overall performance of SSI





## Unreliable Networks

shared-nothing systems: i.e., a bunch of machines connected by a network. The network is the only way those machines can communicate

## why:
- comparatively cheap because it requires no special hardware
- make use of commoditized cloud computing services
- it can achieve high reliability through redundancy across multiple geographically distributed datacenters.


asynchronous packet networks: network gives no guarantees as to when it will arrive, or whether it will arrive at all. (asynchronous networks have unbounded delays)
- request 
    - request lost
    - request may be waiting in a queue and will be delivered later
- remote node
    - remote node may have failed
    - remote node may have temporarily stopped responding
- response
    - response lost
    - response has been delayed and will be delivered later

These issues are indistinguishable in an asynchronous network: the only information you have is that you haven’t received a response yet.

The usual way of handling this issue is a timeout

network problems can be surprisingly common (adding redundant networking gear doesn’t reduce faults as much as you might hope, since it doesn’t guard against human error (e.g., misconfigured switches), which is a major cause of outages.)

reason
- resources are shared among many customers

synchronous network: even as data passes through several routers, it does not suffer from queueing, because it establishes a circuit: a fixed, guaranteed amount of bandwidth is allocated for the call. maximum end-to-end latency of the network is fixed. We call this a bounded delay. (static way)

circuit is a fixed amount of reserved bandwidth. whereas the packets of a TCP connection opportunistically use whatever network bandwidth is available.

internet are optimized for bursty traffic. common internet behavior doesn’t have any particular bandwidth requirement—we just want it to complete as quickly as possible. Thus, using circuits for bursty data transfers wastes network capacity and makes transfers unnecessarily slow. By contrast, TCP dynamically adapts the rate of data transfer to the available network capacity. (dynamically, maximizes utilization of the wire, similar to CPU)

- feature of data: predictable or not
- max utilization

Variable delays in networks are not a law of nature, but simply the result of a cost/ benefit trade-off.

### detecting

need to automatically detect faulty nodes:
- A load balancer needs to stop sending requests to a node that is dead
- single-leader replication, if the leader fails, one of the followers needs to be promoted to be the new leader

It's possible to get rapid feedback about a remote node being down. but you can’t count on it. Even if TCP acknowledges that a packet was delivered, the application may have crashed before handling it. If you want to be sure that a request was successful, you need a positive response from the application itself. 

timeout: 
A short timeout detects faults faster, but carries a higher risk of incorrectly declaring a node dead
- the action may end up being performed twice.
- When a node is declared dead, its responsibilities need to be transferred to other nodes, which places additional load on other nodes and the network. (make it worse => cause a cascading failure)

- choose timeouts experimentally: trade-off between failure detection delay and risk of premature timeouts.
- systems can continually measure response times and their variability, automatically adjust timeouts according to the observed response time distribution.

queue (source of delay):
-  network switch must queue if several different nodes simultaneously try to send packets to the same destination (network congestion)
- operating system queue
- virtualized environments paused while another virtual machine uses a CPU core
- TCP performs flow control in which a node limits its own rate of sending in order to avoid overloading a network link (additional queueing at the sender before the data even enters the network.)
- * TCP retry when timeout

## Unreliable Clocks

two kind of situation needing clock
- measure durations
- describe points in time
- * determine order

each machine on the network has its own clock, which is an actual hardware device: usually a quartz crystal oscillator. not perfectly accurate, so each machine has its own notion of time,

It is possible to synchronize clocks to some degree. the most commonly used mechanism is the Network Time Protocol (NTP) (adjusted according to the time reported by a group of servers)

### Monotonic Versus Time-of-Day Clocks

Time-of-day clocks： current date and time according to some calendar (also known as wall-clock time)

synchronized with NTP, if the local clock is too far ahead of the NTP server, it may be forcibly reset and appear to jump back to a previous point in time. These jumps, as well as the fact that they often ignore leap seconds, make time-of-day clocks unsuitable for measuring elapsed time

Monotonic clocks: 
- suitable for measuring a duration, 
- they are guaranteed to always move forward
- the absolute value of the clock is meaningless
NTP may adjust the frequency at which the monotonic clock moves forward (this is known as slewing the clock). but NTP cannot cause the monotonic clock to jump forward or backward.

### Clock Synchronization and Accuracy

it could be inaccurate
- The quartz clock in a computer is not very accurate: it drifts. This drift limits the best possible accuracy you can achieve
- If a computer’s clock differs too much from an NTP server, it may refuse to synchronize, or the local clock will be forcibly reset (=> see time go backward or suddenly jump forward.)
- If a node is accidentally firewalled off from NTP servers, the misconfiguration may go unnoticed for some time.
- NTP synchronization can only be as good as the network delay
- Some NTP servers are wrong or misconfigured, but they query several servers and ignore outliers. Nevertheless, it’s somewhat worrying to bet the correctness of your systems on the time that you were told by a stranger on the internet.
- messes up timing assumptions in systems that are not designed with leap seconds
- In virtual machines, the hardware clock is virtualized, When a CPU core is shared between virtual machines, this pause manifests itself as the clock suddenly jumping forward
- If you run software on devices that you don’t fully control. you probably cannot trust the device’s hardware clock at all.

It is possible to achieve very good clock accuracy, but it's expensive.

### detecting
robust software needs to be prepared to deal with incorrect clocks.

problem: incorrect clocks easily go unnoticed. if its quartz clock is defective or its NTP client is misconfigured, most things will seem to work fine. result is more likely to be silent and subtle data loss than a dramatic crash.
- carefully monitor the clock offsets between all the machines
- Any node whose clock drifts too far from the others should be declared dead and removed from the cluster.


### Timestamps for ordering events

it is tagged with a timestamp according to the time-of-day clock on the node where the write originated.

the timestamps in Figure 8-3 fail to order the events correctly

This conflict resolution strategy is called last write wins (LWW). Problem: 
- sender's clock is inaccurate, network delay may make order of occurred sequentially reversed
- cannot distinguish between writes that occurred sequentially in quick succession and writes that were truly concurrent
- same timestamp

solve: version vectors

### Clock readings have a confidence interval 

it is more like a range of times

If you’re getting the time from a server, the uncertainty is based on the expected quartz drift since your last sync with the server, plus the NTP server’s uncertainty, plus the network round-trip time to the server

Google’s TrueTime API in Spanner reports the confidence interval on the local clock [earliest, latest].the clock knows that the actual current time is somewhere within that interval. 

### Synchronized clocks for global snapshots

a global, monotonically increasing transaction ID (across all partitions) is difficult to generate。 creating transaction IDs in a distributed system becomes an untenable bottleneck。 The problem, of course, is the uncertainty about clock accuracy.

Spanner implements snapshot isolation across datacenters in this way [59, 60]. It uses the clock’s confidence interval as reported by the TrueTime API. you could know the order if those two intervals do not overlap.

Spanner deliberately waits for the length of the confidence interval before committing a read-write transaction. By doing so, it ensures that any transaction that may read the data is at a sufficiently later time, so their confidence intervals do not overlap.

## Process pause

One option is for the leader to obtain a lease from the other nodes. 

lease contains expiry time
problem:
- relying on synchronized clocks: the expiry time on the lease is set by a different machine
- unexpected pause in the execution of the program (there is nothing to tell this thread that it was paused for so long)

reason of pause
- garbage collector (GC)
- In virtualized environments, a virtual machine can be suspended (pausing the execution and saving the contents of memory to disk) and resumed (restoring the contents of memory and continuing execution)
- end-user devices such as laptops, execution may also be suspended and resumed arbitrarily
- operating system context-switches to another thread
- be paused waiting for a slow disk I/O operation to complete
- swapping to disk (paging). page fault that requires a page from disk to be loaded into memory
- A Unix process can be paused by sending it the SIGSTOP signal

When writing multi-threaded code on a single machine, we have fairly good tools for making it thread-safe. Unfortunately, these tools don’t directly translate to distributed systems, because a distributed system has no shared memory—only messages sent over an unreliable network.

A node in a distributed system must assume that its execution can be paused for a significant length of time at any point, even in the middle of a function.

### handle process pause

Those reasons for pausing can be eliminated if you try hard enough.
- hard real-time systems. 
    - carefully designed and tested to meet specified timing guarantees in all circumstances.
    - Providing real-time guarantees in a system requires support from all levels of the software stack
    - developing real-time systems is very expensive, and they are most commonly used in safety-critical embedded devices.
    - real-time guarantees are simply not economical or appropriate
- Limiting the impact of garbage collection (reduce their impact on the application.)
    - Language runtimes have some flexibility around when they schedule garbage collections, because they can track the rate of object allocation and the remaining free memory over time.
    - treat GC pauses like brief planned outages of a node, and to let other nodes handle requests from clients while one node is collecting its garbage.
    - sending new requests to that node, wait for it to finish processing outstanding requests, and then perform the GC while no requests are in progress. (hides GC pauses from clients)
    - use the garbage collector only for short-lived objects (which are fast to collect) and to restart processes periodically, before they accumulate enough long-lived objects to require a full GC of long-lived objects

## Knowledge, Truth, and Lies

### The Truth Is Defined by the Majority

The moral of these stories is that a node cannot necessarily trust its own judgment of a situation. A distributed system cannot exclusively rely on a single node

Instead, many distributed algorithms rely on a quorum, that is, voting among the nodes: decisions require some minimum number of votes from several nodes in order to reduce the dependence on any one particular node.

The leader and the lock

Frequently, a system requires there to be only one of some thing
- one leader
- one holding lock

Implementing this in a distributed system requires care: even if a node believes that it is “the chosen one” that doesn’t necessarily mean a quorum of nodes agrees!

When the paused client comes back, it believes (incorrectly) that it still has a valid lease and proceeds to also write to the file.

#### Fencing tokens

When the lock server grants a lock or lease, it also returns a fencing token. We can then require that every time a client sends a write request to the storage service, it must include its current fencing token. So that server could reject request if a newer one has already been processed.

idea: it is unwise for a service to assume that its clients will always be well behaved, because the clients are often run by people whose priorities are very different from the priorities of the people running the service [76]. Thus, it is a good idea for any service to protect itself from accidentally abusive clients.

### Byzantine Faults

A system is Byzantine fault-tolerant if it continues to operate correctly even if some of the nodes are malfunctioning and not obeying the protocol, or if malicious attackers

#### Weak forms of lying

### System Model and Reality

- timing assumptions
    - Synchronous model
    - Partially synchronous model
    - Asynchronous model
- node failures
    - Crash-stop faults
    - Crash-recovery faults
    - Byzantine (arbitrary) faults

- correctness
    - Safety
    - liveness




# Linearizability

strongest consistency models, pros & cons


## def: 
make a system appear as if there were only one copy of the data, and all operations on it are atomic


as soon as one client successfully completes a write, all clients reading from the database must be able to see the value just written. guaranteeing that the value read is the most recent, up-to-date value, and doesn’t come from a stale cache or replica.

recency guarantee

there must be some point in time (between the start and end of the write operation) at which the value of x atomically flips from 0 to 1. After any one read has returned the new value, all following reads (on the same or other clients) must also return the new value.

## Serializability vs Linearizability
- Serializability is an isolation property of transactions
- Linearizability is a recency guarantee on reads and writes of a register (an individual object). It doesn’t group operations together into transactions, (not provide Serializability)
- serializable snapshot isolation, (does not include writes that are more recent, so read is not linearizable)


## what replay on Linearizability

### Locking and leader election

use lock to select leader. who get the lock become the leader. 

use lock to enable disk being shared among multiple nodes

### Constraints and uniqueness guarantees

username
similar to a lock
similar to an atomic compare-and-set

bank account balance never goes negative

### Cross-channel timing dependencies

The linearizability violation was only noticed because there was an additional communication channel in the system

send messga after ack from store img.

The linearizability violation was only noticed because there was an additional communication channel in the system

1. replication systems that are linearizable or not
    1. Single-leader replication (potentially linearizable)
    2. Consensus algorithms (linearizable)
    3. Multi-leader replication (not linearizable)
    4. Leaderless replication (probably not linearizable)


## The Cost of Linearizability

single-leader vs multiple-leader for for multidatacenter replication

### CAP

1. requires linearizability, and some replicas are disconnected from the other replicas due to a network problem, then some replicas cannot process requests while they are disconnected - unavailable
2. if available even if it is disconnected from other replicas. its behavior is not linearizable


pick 2 out of 3. Unfortunately, putting it this way is misleading [32] because network partitions are a kind of fault, so they aren’t something about which you have a choice: they will happen whether you like it or not

At times when the network is working correctly, a system can provide both consistency (linearizability) and total availability

When a network fault occurs, you have to choose between either linearizability or total availability.

The CAP theorem as formally defined [30] is of very narrow scope: it only considers one consistency model (namely linearizability) and one kind of fault (network partitions, vior nodes that are alive but disconnected from each other).

### Linearizability and network delays

even RAM on a modern multi-core CPU is not linearizable. It makes no sense to use the CAP theorem. The reason for dropping linearizability is performance, not fault tolerance.

prove that if you want linearizability, the response time of read and write requests is at least proportional to the uncertainty of delays in the network.

but weaker consistency models can be much faster, so this trade-off is important for latency-sensitive systems.

# Ordering Guarantees

where our discussion involved order
- main purpose of the leader in single-leader replication is to determine the order of writes in the replication log
- order in Serializability
- use of timestamps and clocks to determine which one of two writes happened later.

## Ordering and Causality

There are several reasons why ordering keeps coming up, and one of the reasons is that it helps preserve causality.
- our intuition of cause and effect - causal dependency
- Example in replication. Causality here means that a row must first be created before it can be updated.
- have two operations A and B, there are three possibilities: either A happened before B, or B happened before A, or A and B are concurrent (This happened before relationship is another expression of causality) - if A happened before B, that means B might have known about A, or built upon A, or depended on A.
- snapshot isolation: a transaction reads from a consistent snapshot. (consistent with causality) (Observing the entire database at a single point in time makes it consistent with causality) 
- write skew (go off call ) Alice was allowed to go off call because the transaction thought that Bob was still on call, and vice versa. (detects write skew by tracking the causal dependencies between transactions.)
- score: Alice’s exclamation is causally dependent on the announcement of the score

Causality imposes an ordering on events: cause comes before effect. These chains of causally dependent operations define the causal order in the system. If a system obeys the ordering imposed by causality, we say that it is causally consistent.

when you read from the database, and you see some piece of data, then you must also be able to see any data that causally precedes it

### The causal order is not a total order

A total order allows any two elements to be compared
- natural numbers are totally ordered
- Linearizability
partially ordered: in some cases one set is greater than another (if one set contains all the elements of another), but in other cases they are incomparable. 
- mathematical sets are not totally ordered: is {a, b} greater than {b, c}? Well, you can’t really compare them, because neither is a subset of the other. We say they are incomparable
- Causality

Concurrency would mean that the timeline branches and merges again

### Linearizability is stronger than causal consistency

relationship: linearizability implies causality

if there are multiple communication channels. linearizability ensures that causality is automatically preserved without the system having to do anything special

causal consistency is the strongest possible consistency model that does not slow down due to network delays, and remains available in the face of network failures

### Capturing causal dependencies

which operation happened before which other operation

If a node had already seen the value X when it issued the write Y, then X and Y may be causally related

it needs to track causal dependencies across the entire database, not just for a single key. Version vectors can be generalized to do this

know which version of the data was read by the application

A similar idea appears in the conflict detection of SSI, as discussed in “Serializable Snapshot Isolation (SSI)” on page 261: when a transaction wants to commit, the database checks whether the version of the data that it read is still up to date. To this end, the database keeps track of which data has been read by which transaction.

## Sequence Number Ordering

actually keeping track of all causal dependencies can become impractical. In many applications, clients read lots of data before writing something, and then it is not clear whether the write is causally dependent on all or only some of those prior reads.

we can use sequence numbers or timestamps to order events.

every operation has a unique sequence number, and you can always compare two sequence numbers

we can create sequence numbers in a total order that is consistent with causality

### Noncausal sequence number generators

### Lamport timestamps

Each node has a unique identifier, and each node keeps a counter of the number of operations it has processed. The Lamport timestamp is then simply a pair of (counter, node ID). 

if you have two timestamps, the one with a greater counter value is the greater timestamp; if the counter values are the same, the one with the greater node ID is the greater timestamp.
- every node and every client keeps track of the maximum counter value it has seen so far, and includes that maximum on every request.
- When a node receives a request or response with a maximum counter value greater than its own counter value, it immediately increases its own counter to that maximum.

every causal dependency results in an increased timestamp.

#### vs version vector

version vectors can distinguish whether two operations are concurrent or whether one is causally dependent on the other, whereas Lamport timestamps always enforce a total ordering.

The advantage of Lamport timestamps over version vectors is that they are more compact.

### Timestamp ordering is not sufficient

This approach works for determining the winner after the fact: once you have collected all the username creation operations in the system, you can compare their timestamps.

However, it is not sufficient when a node has just received a request from a user to create a username, and needs to decide right now whether the request should succeed or fail.

At that moment, the node does not know whether another node is concurrently in the process of creating an account with the same username, and what timestamp that other node may assign to the operation.

In order to be sure that no other node is in the process of concurrently creating an account with the same username and a lower timestamp, you would have to check with every other node to see what it is doing

The problem here is that the total order of operations only emerges after you have collected all of the operations.

## Total Order Broadcast

getting all nodes to agree on the same total ordering of operations is tricky

As discussed, single-leader replication determines a total order of operations by choosing one node as the leader and sequencing all operations on a single CPU core on the leader.

how to scale the system if the throughput is greater than a single leader can handle

Total order broadcast is usually described as a protocol for exchanging messages between nodes. Informally, it requires that two safety properties always be satisfied:
- Reliable delivery
    No messages are lost: if a message is delivered to one node, it is delivered to all nodes.
- Totally ordered delivery 
    Messages are delivered to every node in the same order.

A correct algorithm for total order broadcast must ensure that the reliability and ordering properties are always satisfied, even if a node or the network is faulty.

an algorithm can keep retrying so that the messages get through when the network is eventually repaired 

### Using total order broadcast

This fact is a hint that there is a strong connection between total order broadcast and consensus

Total order broadcast is exactly what you need for database replication: if every message represents a write to the database, and every replica processes the same writes in the same order, then the replicas will remain consistent with each other (aside from any temporary replication lag). This principle is known as state machine replication

implement serializable transactions: every message represents a deterministic transaction to be executed as a stored procedure, and if every node processes those messages in the same order, then the partitions and replicas of the database are kept consistent with each other

Another way of looking at total order broadcast is that it is a way of creating a log (as in a replication log, transaction log, or write-ahead log): delivering a message is like appending to the log.

Total order broadcast is also useful for implementing a lock service that provides fencing tokens.

acquire the lock is appended as a message to the log, and all messages are sequentially numbered in the order they appear in the log.

### Implementing linearizable storage using total order broadcast

using total order broadcast as an append-only log

1. Append a message to the log, tentatively indicating the username you want to claim.

2. Read the log, and wait for the message you appended to be delivered back to you.xi 

3. Check for any messages claiming the username that you want. If the first message for your desired username is your own message, then you are successful

Because log entries are delivered to all nodes in the same order, if there are several concurrent writes, all nodes will agree on which one came first. Choosing the first of the conflicting writes as the winner and aborting later ones

While this procedure ensures linearizable writes, it doesn’t guarantee linearizable reads - if you read from a store that is asynchronously updated from the log

sequence reads through the log by appending a message, reading the log, and performing the actual read when the message is delivered back to you. The message’s position in the log thus defines the point in time at which the read happens.

If the log allows you to fetch the position of the latest log message in a linearizable way, you can query that position, wait for all entries up to that position to be delivered to you, and then perform the read.

You can make your read from a replica that is synchronously updated on writes

### Implementing total order broadcast using linearizable storage

linearizable register that stores an integer and that has an atomic increment-and-get operation

for every message you want to send through total order broadcast, you increment-and-get the linearizable integer, and then attach the value you got from the register as a sequence number to the message. You can then send the message to all nodes (resending any lost messages), and the recipients will deliver the messages consecutively by sequence number.

numbers you get from incrementing the linearizable register form a sequence with no gaps.

fault-tolerance

# Distributed Transactions and Consensus

Consensus: get several nodes to agree on something.

situations in which it is important for nodes to agree
- Leader election
- Atomic commit: get all nodes to agree on the outcome of the transaction: either they all abort/roll back

## Atomic Commit and Two-Phase Commit (2PC)

### distributed atomic commit

For example, perhaps you have a multi-object transaction in a partitioned database. it is not sufficient to simply send a commit request to all. commit succeeds on some nodes and fails on other nodes
- detect a constraint violation or conflict, making an abort necessary, while other nodes are successfully able to commit.
- commit requests might be lost in the network, eventually aborting due to a timeout, while other commit requests get through.
- crash before the commit record is fully written and roll back on recovery, while others successfully commit.

A transaction commit must be irrevocable:
- once data has been committed, it becomes visible to other transactions, and thus other clients may start relying on that data

### Introduction to two-phase commit

2PC uses a new component that does not normally appear in single-node transactions: a coordinator
We call these database nodes participants in the transaction.

txn id to identify

phase 1: it sends a prepare request to each of the nodes, asking them whether they are able to commit.

When a participant receives the prepare request, it makes sure that it can definitely commit the transaction under all circumstances. This includes writing all transaction data to disk (a crash, a power failure, or running out of disk space is not an acceptable excuse for refusing to commit later), and checking for any conflicts or constraint violations.

phase 2:
- If all participants reply “yes,” then the coordinator sends out a commit request
- If any of the participants replies “no,” sends an abort request

The coordinator must write that decision to its transaction log on disk so that it knows which way it decided in case it subsequently crashes. This is called the commit point.

Once the coordinator’s decision has been written to disk, the commit or abort request is sent to all participants.

retry forever until it succeeds

two crucial “points of no return”:
1. when a participant votes “yes,” it promises that it will definitely be able to commit later
2. oordinator decides, that decision is irrevocable

### Coordinator failure

the coordinator crashed before it could send the commit request
If the coordinator crashes or the network fails at this point, the participant can do nothing but wait. A participant’s transaction in this state is called in doubt or uncertain.

commit / abort timeout doesn't help

The only way 2PC can complete is by waiting for the coordinator to recover.

This is why the coordinator must write its commit or abort decision to a transaction log on disk before sending commit or abort requests to participants

#### Three-phase commit (?)

Two-phase commit is called a blocking atomic commit protocol due to the fact that 2PC can become stuck waiting for the coordinator to recover.

3PC assumes a network with bounded delay and nodes with bounded response times

In general, nonblocking atomic commit requires a perfect failure detector

### Distributed Transactions in Practice

pro: providing an important safety guarantee that would be hard to achieve otherwise

con: criticized for causing operational problems, killing performance

Much of the performance cost inherent in two-phase commit is due to the additional disk forcing (fsync) that is required for crash recovery, and the additional network round-trips.

Two quite different types of distributed transactions are often conflated:
1. Database-internal distributed transactions
    all the nodes participating in the transaction are running the same database software.
2. Heterogeneous distributed transactions
    participants are two or more different technologies. even though the systems may be entirely different under the hood.

### Exactly-once message processing (?)

For example, a message from a message queue can be acknowledged as processed if and only if the database transaction for processing the message was successfully committed.
atomically committing the message acknowledgment and the database writes in a single transaction.

Such a distributed transaction is only possible if all systems affected by the transaction are able to use the same atomic commit protocol. 

detail in future

### XA transactions

X/Open XA (short for eXtended Architecture) 
???

X/Open XA (short for eXtended Architecture) is a standard for implementing two-phase commit across heterogeneous technologies

—it is merely a C API

XA assumes that your application uses a network driver or client library to communicate with the participant databases or messaging services. If the driver supports XA, that means it calls the XA API to find out whether an operation should be part of a distributed transaction—and if so, it sends the necessary information to the database server. The driver also exposes callbacks through which the coordinator can ask the participant to prepare, commit, or abort.

### Holding locks while in doubt

Why do we care so much about a transaction being stuck in doubt? The problem is with locking. The database cannot release those locks until the transaction commits or aborts

### Recovering from coordinator failure

orphaned in-doubt transactions do occur - cannot decide the outcome for whatever reason (because log has been lost or corrupted due to a software bug) These transactions cannot be resolved automatically, so they sit forever in the database, holding locks and blocking other transactions.

Even rebooting your database servers will not fix this problem, since a correct implementation of 2PC must preserve the locks of an in-doubt transaction even across restarts

The only way out is for an administrator to manually decide

Many XA implementations have an emergency escape hatch called heuristic decisions: allowing a participant to unilaterally decide to abort or commit an in-doubt transaction without a definitive decision from the coordinator

probably breaking atomicity, since it violates the system of promises in two-phase commit.

### Limitations of distributed transactions

- If the coordinator is not replicated but runs only on a single machine, it is a single point of failure for the entire system
- Many server-side applications are developed in a stateless model (as favored by HTTP), with all persistent state stored in a database (benefit: application servers can be added and removed at will). However, when the coordinator is part of the application server, it changes the nature of the deployment. Suddenly, the coordinator’s logs become a crucial part. Such application servers are no longer stateless.
- XA needs to be compatible with a wide range of data systems, it is necessarily a lowest common denominator.
    - cannot detect deadlocks across different systems (since that would require a standardized protocol for systems to exchange information on the locks
    - does not work with SSI, since that would require a protocol for identifying conflicts across different systems.
- there remains the problem that for 2PC to successfully commit a transaction: all participants must respond. if any part of the system is broken, the transaction also fails. Distributed transactions thus have a tendency of amplifying failures,



## Fault-Tolerant Consensus

The consensus problem is normally formalized as follows: one or more nodes may propose values, and the consensus algorithm decides on one of those values.

In this formalism, a consensus algorithm must satisfy the following properties [25]:xiii 
- Uniform agreement 
    No two nodes decide differently. 
- Integrity
    No node decides twice.
- Validity 
    If a node decides value v, then v was proposed by some node. 
- Termination
    Every node that does not crash eventually decides some value.

uniform agreement and integrity properties define the core idea of consensus: everyone decides on the same outcome, and once you have decided, you cannot change your mind.

The validity property exists mostly to rule out trivial solutions: for example, you could have an algorithm that always decides null, no matter what was proposed;

If you don’t care about fault tolerance, then satisfying the first three properties is easy: you can just hardcode one node to be the “dictator,”

The termination property formalizes the idea of fault tolerance. Even if some nodes fail, the other nodes must still reach a decision. (The system model of consensus assumes that when a node “crashes,” it suddenly disappears and never comes back. any algorithm that has to wait for a node to recover is not going to be able to satisfy the termination property. In particular, 2PC does not meet the requirements for termination.)

a limit to the number of failures that an algorithm can tolerate: in fact, it can be proved that any consensus algorithm requires at least a majority of nodes to be functioning correctly in order to assure termination

However, most implementations of consensus ensure that the safety properties—agreement, integrity, and validity—are always met, even if a majority of nodes fail or there is a severe network problem [92]. Thus, a large-scale outage can stop the system from being able to process requests, but it cannot corrupt the consensus system by causing it to make invalid decisions.

Most consensus algorithms assume that there are no Byzantine faults

robust against Byzantine faults as long as fewer than one-third of the nodes are Byzantine-faulty

### Consensus algorithms and total order broadcast

Instead, they decide on a sequence of values, which makes them total order broadcast algorithms

If you think about it, this is equivalent to performing several rounds of consensus: in each round, nodes propose the message that they want to send next, and then decide on the next message to be delivered in the total order

- Due to the agreement property of consensus, all nodes decide to deliver the same messages in the same order.
- Due to the integrity property, messages are not duplicated.
- Due to the validity property, messages are not corrupted and not fabricated out of thin air.
- Due to the termination property, messages are not lost.

### Single-leader replication and consensus (?)

essentially have a “consensus algorithm” of the dictatorial variety: only one node is allowed to accept writes

Some databases perform automatic leader election and failover, promoting a follower to be the new leader if the old leader fails

we need consensus in order to elect a leader.

### Epoch numbering and quorums

they don’t guarantee that the leader is unique. Instead, they can make a weaker guarantee: the protocols define an epoch number and guarantee that within each epoch, the leader is unique.

This election is given an incremented epoch number, and thus epoch numbers are totally ordered and monotonically increasing.

leader with the higher epoch number prevails.

Before a leader is allowed to decide anything, it must first check that there isn’t some other leader with a higher epoch number - a node cannot necessarily trust its own judgment

it must collect votes from a quorum of nodes

wait for a quorum of nodes to respond in favor of the proposal. The quorum typically, but not always, consists of a majority of nodes

we have two rounds of voting: once to choose a leader, and a second time to vote on a leader’s proposal

he quorums for those two votes must overlap: if a vote on a proposal succeeds, at least one of the nodes that voted for it must have also participated in the most recent leader election

if the vote on a proposal does not reveal any higher-numbered epoch, the current leader can conclude that no leader election with a higher epoch number has happened

The biggest differences are that in 2PC the coordinator is not elected, and that fault-tolerant consensus algorithms only require votes from a majority of nodes, whereas 2PC requires a “yes” vote from every participant.

consensus algorithms define a recovery process by which nodes can get into a consistent state after a new leader is elected, ensuring that the safety properties are always met.


### Limitations of consensus

- nodes vote on proposals before they are decided is a kind of synchronous replication.
- Consensus systems always require a strict majority to operate. 
- a fixed set of nodes that participate in voting, which means that you can’t just add or remove nodes in the cluster. Dynamic membership extensions to consensus algorithms allow the set of nodes in the cluster to change over time, but they are much less well understood than static membership algorithms.
- rely on timeouts to detect failed nodes. frequent leader elections result in terrible performance
- consensus algorithms are particularly sensitive to network problems. (if the entire network is working correctly except for one particular network link that is consistently unreliable, Raft can get into situations where leadership continually bounces between two nodes, or the current leader is continually forced to resign)

## Membership and Coordination Services

ZooKeeper implementing not only total order broadcast (and hence consensus), but also an interesting set of other features:
- Linearizable atomic operations
    Using an atomic compare-and-set operation, you can implement a lock / lease
- Total ordering of operations
    fencing token - totally ordering all operations and giving each operation a monotonically increasing transaction ID (zxid)
- Failure detection
    periodically exchange heartbeats to check that the other node is still alive.
- Change notifications
    subscribing to notifications, a client avoids having to frequently poll to find out about changes.

Of these features, only the linearizable atomic operations really require consensus. However, it is the combination of these features that makes systems like ZooKeeper so useful for distributed coordination.

### Allocating work to nodes

which partition to assign to which node. As new nodes join the cluster, some of the partitions need to be moved from existing nodes to the new nodes in order to rebalance

by judicious use of atomic operations, ephemeral nodes, and notifications in ZooKeeper.

got: automatically recover from faults without human intervention

perform majority votes over so many nodes would be terribly inefficient

?? runs on a fixed number of nodes (usually three or five) and performs its majority votes among those nodes while supporting a potentially large number of clients. Thus, ZooKeeper provides a way of “outsourcing” some of the work of coordinating nodes to an external service.


### Service discovery (???)

ZooKeeper, etcd, and Consul are also often used for service discovery. which IP address you need to connect to in order to reach a particular service.

it is less clear whether service discovery actually requires consensus. DNS is the traditional way of looking up the IP address for a service name, and it uses multiple layers of caching to achieve good performance and availability.

Although service discovery does not require consensus, leader election does. Thus, if your consensus system already knows who the leader is, then it can make sense to also use that information to help other services discover who the leader is.

consensus systems support read-only caching replicas. These replicas asynchronously receive the log of all decisions of the consensus algorithm, but do not actively participate in voting. They are therefore able to serve read requests that do not need to be linearizable.

### Membership services

ZooKeeper and friends can be seen as part of a long history of research into membership services, A membership service determines which nodes are currently active and live members of a cluster.

it’s not possible to reliably detect whether another node has failed. However, if you couple failure detection with consensus, nodes can come to an agreement about which nodes should be considered alive or not.

nevertheless very useful for a system to have agreement on which nodes constitute the current membership.



# Chapter 10 Batch processing

three different types of systems:
- Services (online systems)
    - A service waits for a request or instruction from a client to arrive. When one is received, the service tries to handle it as quickly as possible and sends a response back. 
    - Response time is usually the primary measure of performance.
- Batch processing systems (offline systems)
    - A batch processing system takes a large amount of input data, runs a job to process it, and produces some output data. Jobs often take a while, so there normally isn’t a user waiting for the job to finish. Instead, batch jobs are often scheduled to run periodically. 
    - Throughput is usually the primary measure of performance.
- Stream processing systems (near-real-time systems)
    - Stream processing is somewhere between online and offline/batch processing (so it is sometimes called near-real-time or nearline processing).
    - Like a batch processing system, a stream processor consumes inputs and produces outputs (rather than responding to requests).
    - However, a stream job operates on events shortly after they happen, whereas a batch job operates on a fixed set of input data.

MapReduce is a fairly low-level programming model compared to the parallel processing systems (e.g. MPP), but it was a major step forward in terms of the scale of processing that could be achieved on commodity hardware.

## Batch Processing with Unix Tools
### The Unix Philosophy
1. Make each program do one thing well. To do a new job, build afresh rather than complicate old programs by adding new “features”.
2. Expect the output of every program to become the input to another, as yet unknown, program. Don’t clutter output with extraneous information.
3. Design and build software, even operating systems, to be tried early, ideally within weeks. Don’t hesitate to throw away the clumsy parts and rebuild them.
4. Use tools in preference to unskilled help to lighten a programming task, even if you have to detour to build the tools and expect to throw some of them out after you’ve finished using them.

This approach—automation, rapid prototyping, incremental iteration, being friendly to experimentation, and breaking down large projects into manageable chunks is still in-trend today

A Unix shell like bash lets us easily compose these small programs into surprisingly powerful data processing jobs. What does Unix do to enable this composability?

### A uniform interface

those programs must use the same data format—in other words, a compatible interface. 
all programs must use the same input/output interface.

In Unix, that interface is a file (or, more precisely, a file descriptor). A file is just an ordered sequence of bytes. Because that is such a simple interface, many different things can be represented using the same interface.

By convention, many (but not all) Unix programs treat this sequence of bytes as ASCII text, treat their input file as a list of records separated by the \n. all these programs have standardized on using the same record separator allows them to interoperate.

### Separation of logic and wiring

Another characteristic feature of Unix tools is their use of standard input (stdin) and standard output (stdout).

Pipes let you attach the stdout of one process to the stdin of another process (with a small in-memory buffer, and without writing the entire intermediate data stream to disk).

This allows a shell user to wire up the input and output in whatever way they want. the program doesn’t know or care where the input is coming from and where the output is going to. (One could say this is a form of loose coupling, late binding [15], or inversion of control [16].)

limit: Programs that need multiple inputs or outputs are possible but tricky. You can’t pipe a program’s output into a network connection

### Transparency and experimentation

- The input files to Unix commands are normally treated as immutable. So re-running won't damage the input files
- You can end the pipeline at any point, pipe the output into less, and look at it to see if it has the expected form. This ability to inspect is great for debugging.
- You can write the output of one pipeline stage to a file and use that file as input to the next stage. This allows you to restart the later stage without rerunning the entire pipeline.

## MapReduce and Distributed Filesystems

As with most Unix tools, running a MapReduce job normally does not modify the input and does not have any side effects other than producing the output.

The output files are written once, in a sequential fashion (not modifying any existing part of a file once it has been written).

MapReduce jobs read and write files on a distributed filesystem. In Hadoop’s implementation of MapReduce, that filesystem is called HDFS (Hadoop Distributed File System), an open source reimplementation of the Google File System (GFS).

HDFS is based on the shared-nothing principle (see the introduction to Part II), in contrast to the shared-disk approach of Network Attached Storage (NAS) and Storage Area Network (SAN) architectures. the shared-nothing approach requires no special hardware, only computers connected by a conventional datacenter network. 

HDFS consists of a daemon process running on each machine, exposing a network service that allows other nodes to access files. A central server called the NameNode keeps track of which file blocks are stored on which machine. Thus, HDFS conceptually creates one big filesystem that can use the space on the disks of all machines running the daemon.

In order to tolerate machine and disk failures, file blocks are replicated on multiple machines. Replication may mean simply several copies of the same data on multiple machines, as in Chapter 5, or an erasure coding scheme such as Reed–Solomon codes, which allows lost data to be recovered with lower storage overhead than full replication.

# MapReduce Job Execution

1. Read a set of input files, and break it up into records. 
In the web server log example, each record is one line in the log (that is, \n is the record separator).

2. Call the mapper function to extract a key and value from each input record. 
In the preceding example, the mapper function is awk '{print $7}': it extracts the URL ($7) as the key, and leaves the value empty.

3. Sort all of the key-value pairs by key. 
In the log example, this is done by the first sort command.

4. Call the reducer function to iterate over the sorted key-value pairs. If there are multiple occurrences of the same key, the sorting has made them adjacent in the list, so it is easy to combine those values without having to keep a lot of state in memory. 
In the preceding example, the reducer is implemented by the command uniq -c, which counts the number of adjacent records with the same key.

Steps 2 (map) and 4 (reduce) are where you write your custom data processing code. Step 1 (breaking files into records) is handled by the input format parser. Step 3, the sort step, is implicit in MapReduce—you don’t have to write it, because the output from the mapper is always sorted before it is given to the reducer.

Mapper
The mapper is called once for every input record, and its job is to extract the key and value from the input record. For each input, it may generate any number of key-value pairs (including none). It does not keep any state from one input record to the next, so each record is handled independently.

Reducer
The MapReduce framework takes the key-value pairs produced by the mappers, collects all the values belonging to the same key, and calls the reducer with an iterator over that collection of values. The reducer can produce output records (such as the number of occurrences of the same URL).

## Distributed execution of MapReduce

MapReduce can parallelize a computation across many machines, without you having to write code to explicitly handle the parallelism.

Its parallelization is based on partitioning: the input to a job is typically a directory in HDFS, and each file or file block within the input directory is considered to be a separate partition that can be processed by a separate map task.

run each mapper on one of the machines that stores a replica of the input file, provided that machine has enough spare RAM and CPU resources to run the map task [26]. 
This principle is known as putting the computation near the data [27]: it saves copying the input file over the network, reducing network load and increasing locality.

the MapReduce framework first copies the code (e.g., JAR files in the case of a Java program) to the appropriate machines.

The reduce side of the computation is also partitioned. While the number of map tasks is determined by the number of input file blocks, the number of reduce tasks is configured by the job author (it can be different from the number of map tasks). To ensure that all key-value pairs with the same key end up at the same reducer, the framework uses a hash of the key to determine which reduce task should receive a particular key-value pair

Instead, the sorting is performed in stages. First, each map task partitions its output by reducer, based on the hash of the key. Each of these partitions is written to a sorted file on the mapper’s local disk, using a technique similar to what we discussed in “SSTables and LSMTrees” on page 76.

Whenever a mapper finishes reading its input file and writing its sorted output files, the MapReduce scheduler notifies the reducers that they can start fetching the output files from that mapper. The reducers connect to each of the mappers and download the files of sorted key-value pairs for their partition. 
copying data partitions from mappers to reducers is known as the shuffle [26] (a confusing term—unlike shuffling a deck of cards, there is no randomness in MapReduce).

The reduce task takes the files from the mappers and merges them together, preserving the sort order. Thus, if different mappers produced records with the same key, they will be adjacent in the merged reducer input.

The reducer is called with a key and an iterator that incrementally scans over all records with the same key. The reducer can use arbitrary logic to process these records, and can generate any number of output records. These output records are written to a file on the distributed filesystem (usually, one copy on the local disk of the machine running the reducer, with replicas on other machines).

## MapReduce workflows

it is very common for MapReduce jobs to be chained together into workflows, such that the output of one job becomes the input to the next job. The Hadoop MapReduce framework does not have any particular support for workflows, so this chaining is done implicitly by directory name: the first job must be configured to write its output to a designated directory in HDFS, and the second job must be configured to read that same directory name as its input. From the MapReduce framework’s point of view, they are two independent jobs.

Chained MapReduce jobs are therefore less like pipelines of Unix commands (which using only a small in-memory buffer) and more like a sequence of commands where each command’s output is written to a temporary file, and the next command reads from the temporary file. This design has advantages and disadvantages 

A batch job’s output is only considered valid when the job has completed successfully (MapReduce discards the partial output of a failed job). Therefore, one job in a workflow can only start when the prior jobs—have completed successfully. 

various workflow schedulers for Hadoop have been developed, including Oozie, Azkaban, Luigi, Airflow, and Pinball. Various higher-level tools for Hadoop, such as Pig [30], Hive [31], Cascading [32], Crunch [33], and FlumeJava [34], also set up workflows of multiple MapReduce stages that are automatically wired together appropriately.

## Reduce-Side Joins and Grouping

In many datasets it is common for one record to have an association with another record. A join is necessary whenever you have some code that needs to access records on both sides of that association (both the record that holds the reference and the record being referenced).

Example: analysis of user activity events

logged-in users did on a website (known as activity events or clickstream data). the log of events is the fact table, and the user database is one of the dimensions.

correlate user activity with user profile information. determine which pages are most popular with which age groups. 

The simplest implementation of this join would go over the activity events one by one and query the user database (on a remote server) for every user ID it encounters. very poor performance: 
- the processing throughput would be limited by the *round-trip time* to the database server, 
- the effectiveness of a local cache would depend very much on the distribution of data, 
- running a large number of queries in parallel could easily overwhelm the database

In order to achieve good throughput in a batch process, the computation must be (as much as possible) local to one machine. 

Thus, a better approach would be to take a copy of the user database. put it in the same distributed filesystem. 

## Sort-merge joins

When the MapReduce framework partitions the mapper output by key and then sorts the key-value pairs, the effect is that all the activity events and the user record with the same user ID become adjacent to each other in the reducer input.

The reducer can then perform the actual join logic easily: the reducer function is called once for every user ID. 

processes all of the records for a particular user ID in one go, it only needs to keep one user record in memory at any one time, and it never needs to make any requests over the network. This algorithm is known as a sort-merge join, since mapper output is sorted by key, and the reducers then merge together the sorted lists of records from both sides of the join.

## Bringing related data together in the same place

all the necessary data to perform the join operation for a particular user ID is brought together in the same place: a single call to the reducer. 

Using the MapReduce programming model has separated the physical network communication aspects of the computation (getting the data to the right machine) from the application logic (processing the data once you have it).

Since MapReduce handles all network communication, it also shields the application code from having to worry about partial failures, such as the crash of another node: MapReduce transparently retries failed tasks without affecting the application logic.

## GROUP BY

set up the mappers so that the key-value pairs they produce use the desired grouping key. The partitioning and sorting process then brings together all the records with the same key in the same reducer.

## Handling skew

The pattern of “bringing all records with the same key to the same place” breaks down if there is a very large amount of data related to a single key.

Such disproportionately active database records are known as linchpin objects [38] or hot keys. 

Collecting all activity related to a celebrity (e.g., replies to something they posted) in a single reducer can lead to significant skew (also known as hot spots)—that is, one reducer that must process significantly more records than the others. any subsequent jobs must wait for the slowest reducer to complete before they can start.

For example, the skewed join method in Pig first runs a sampling job to determine which keys are hot.

send any records relating to a hot key to one of several reducers, chosen at random. For the other input to the join, records relating to the hot key need to be replicated to all reducers handling that key. 

This technique spreads the work of handling the hot key over several reducers, which allows it to be parallelized better, at the cost of having to replicate the other join input to multiple reducers.

Hive’s skewed join optimization takes an alternative approach. It requires hot keys to be specified explicitly in the table metadata, and it stores records related to those keys in separate files from the rest. When performing a join on that table, it uses a **mapside join** (see the next section) for the hot keys.

When grouping records by a hot key and aggregating them, you can perform the grouping in two stages. The first MapReduce stage sends records to a random reducer, so that each reducer performs the grouping on a subset of records for the hot key and outputs a more compact aggregated value per key. The second MapReduce job then combines the values from all of the first-stage reducers into a single value per key.

## Map-Side Joins

The reduce-side approach has the advantage that you do not need to make any assumptions about the input data: the mappers can prepare the data to be ready for joining. However, the downside is that all that sorting, copying to reducers, and merging of reducer inputs can be quite expensive.

if you can make certain assumptions about your input data, it is possible to make joins faster by using a so-called map-side join. This approach uses a cut-down MapReduce job in which there are no reducers and no sorting. Instead, each mapper simply reads one input file block from the distributed filesystem and writes one output file to the filesystem—that is all.

### Broadcast hash joins

The simplest way applies in the case where a large dataset is joined with a small dataset. In particular, the small dataset needs to be small enough that it can be loaded entirely into memory in each of the mappers.

In this case, when a mapper starts up, it can first read the user database from the distributed filesystem into an in-memory hash table. Once this is done, the mapper can scan over the user activity events and simply look up the user ID for each event in the hash table.

This simple but effective algorithm is called a broadcast hash join: the word broadcast reflects the fact that each mapper for a partition of the large input reads the entirety of the small input (so the small input is effectively “broadcast” to all partitions of the large input), and the word hash reflects its use of a hash table.

an alternative is to store the small join input in a read-only index on the local disk. The frequently used parts of this index will remain in the operating system’s page cache, so this approach can provide random-access lookups almost as fast as an in-memory hash table, but without actually requiring the dataset to fit in memory.

### Partitioned hash joins

If the inputs to the map-side join are partitioned in the same way, then the hash join approach can be applied to each partition independently. 

activity events and the user database to each be partitioned based on the last decimal digit of the user ID.

If the partitioning is done correctly, you can be sure that all the records you might want to join are located in the same numbered partition, and so it is sufficient for each mapper to only read one partition from each of the input datasets. each mapper can load a smaller amount of data into its hash table.

### Map-side merge joins

input datasets are not only partitioned in the same way, but also sorted based on the same key.

it does not matter whether the inputs are small enough to fit in memory, because a mapper can perform the same merging operation that would normally be done by a reducer: reading both input files incrementally.

If a map-side merge join is possible, it probably means that prior MapReduce jobs brought the input datasets into this partitioned and sorted form in the first place. it may still be appropriate to perform the merge join in a separate maponly job, for example if the partitioned and sorted datasets are also needed for other purposes besides this particular join.

### MapReduce workflows with map-side joins

As discussed, map-side joins also make more assumptions about the size, sorting, and partitioning of their input datasets.

Knowing about the physical layout of datasets in the distributed filesystem becomes important when optimizing join strategies. you must also know the number of partitions and the keys by which the data is partitioned and sorted.

In the Hadoop ecosystem, this kind of metadata about the partitioning of datasets is often maintained in HCatalog and the Hive metastore

## The Output of Batch Workflows

what is the result of all of that processing, once it is done? Why are we running all these jobs in the first place?

It is not transaction processing, nor is it analytics. It is closer to analytics, in that a batch process typically scans over large portions of an input dataset. The output of a batch process is often not a report, but some other kind of structure.

### Building search indexes

how a full-text search index such as Lucene works: it is a file (the term dictionary) in which you can efficiently look up a particular keyword and find the list of all the document IDs containing that keyword (the postings list). This is a very simplified view of a search index—in reality it requires various additional data, in order to rank search results by relevance, correct misspellings, resolve synonyms, and so on—but the principle holds.

the mappers partition the set of documents as needed, each reducer builds the index for its partition, and the index files are written to the distributed filesystem.

If the indexed set of documents changes, one option is to periodically rerun the entire indexing workflow for the entire set of documents, and replace the previous index files wholesale with the new index files when it is done.

### Key-value stores as batch process output

build machine learning systems such as classifiers (e.g., spam filters, anomaly detection, image recognition) and recommendation systems (e.g., people you may know, products you may be interested in, or related searches [29]).

The output of those batch jobs is often some kind of database: for example, a database that can be queried by user ID to obtain suggested friends for that user, or a database that can be queried by product ID to get a list of related products

These databases need to be queried from the web application that handles user requests, which is usually separate from the Hadoop infrastructure. So how does the output from the batch process get back into a database where the web application can query it

The most obvious choice might be to use the client library for your favorite database directly within a mapper or reducer, and to write from the batch job directly to the database server, one record at a time. but it is a bad idea for several reasons:

- As discussed previously in the context of joins, making a network request for every single record is orders of magnitude slower than the normal throughput of a batch task.
- MapReduce jobs often run many tasks in parallel. If all the mappers or reducers concurrently write to the same output database, with a rate expected of a batch process, that database can easily be overwhelmed
- MapReduce provides a clean all-or-nothing guarantee for job output: if a job succeeds, the result is the output of running every task exactly once, even if some tasks failed and had to be retried along the way; if the entire job fails, no output is produced. **externally visible side effects**

A much better solution is to build a brand-new database inside the batch job and write it as files to the job’s output directory in the distributed filesystem. Those data files are then immutable once written, and can be loaded in bulk into servers that handle read-only queries.

When loading data into Voldemort, the server continues serving requests to the old data files while the new data files are copied from the distributed filesystem to the server’s local disk. Once the copying is complete, the server atomically switches over to querying the new files.

### Philosophy of batch process outputs

In the process, the input is left unchanged, any previous output is completely replaced with the new output, and there are no other side effects. This means that you can rerun a command as often as you like, tweaking or debugging it, without messing up the state of your system.

The handling of output from MapReduce jobs follows the same philosophy. By treating inputs as immutable and avoiding side effects (such as writing to external databases), batch jobs not only achieve good performance but also become much easier to maintain:

- If you introduce a bug, simply roll back to a previous version of the code and rerun the job, and the output will be correct again. keep the old output in a different directory and simply switch back to it. Databases with read-write transactions do not have this property. buggy code that writes bad data to the database, then rolling back the code will do nothing to fix the data in the database.
- As a consequence of this ease of rolling back, feature development can proceed more quickly than in an environment where mistakes could mean irreversible damage. This principle of minimizing irreversibility is beneficial for Agile software development
- If a map or reduce task fails, the MapReduce framework automatically reschedules it and runs it again on the same input. if the failure is due to a transient issue, the fault is tolerated. This automatic retry is only safe because inputs are immutable and outputs from failed tasks are discarded by the MapReduce framework.
- The same set of files can be used as input for various different jobs
- Like Unix tools, MapReduce jobs separate logic from wiring (configuring the input and output directories), which provides a separation of concerns and enables potential reuse of code: one team can focus on implementing a job that does one thing well, while other teams can decide where and when to run that job.

Unix and Hadoop also differ in some ways:
- most Unix tools assume untyped text files, they have to do a lot of input parsing. On Hadoop, some of those low-value syntactic conversions are eliminated by using more structured file formats. 

## Comparing Hadoop to Distributed Databases

Hadoop is somewhat like a distributed version of Unix, where HDFS is the filesystem and MapReduce is a quirky implementation of a Unix process.

When the MapReduce paper [1] was published, it was—in some sense—not at all new. All of the processing and parallel join algorithms that we discussed in the last few sections had already been implemented in so-called massively parallel processing (MPP) databases

- MPP databases focus on parallel execution of analytic SQL queries on a cluster of machines
- the combination of MapReduce and a distributed filesystem [19] provides something much more like a general-purpose operating system that can run arbitrary programs.

## Diversity of storage

Hadoop opened up the possibility of indiscriminately dumping data into HDFS, and only later figuring out how to process it further. By contrast, MPP databases typically require careful up-front modeling of the data and query patterns before importing the data into the database’s proprietary storage format.

simply making data available quickly is often more valuable than trying to decide on the ideal data model up front [54].

speed up data collection

Simply dumping data in its raw form allows for several such transformations. This approach has been dubbed the sushi principle: “raw data is better”

### Diversity of processing models

MPP databases are monolithic, tightly integrated pieces of software that take care of storage layout on disk, query planning, scheduling, and execution. Since these components can all be tuned and optimized for the specific needs of the database, the system as a whole can achieve very good performance on the types of queries for which it is designed.

Moreover, the SQL query language allows expressive queries

On the other hand, not all kinds of processing can be sensibly expressed as SQL queries.

need a more general model of data processing

Having two processing models, SQL and MapReduce, was not enough

### Designing for frequent faults

Batch processes are less sensitive to faults than online systems, because they do not immediately affect users if they fail and they can always be run again.

If a node crashes while a query is executing, most MPP databases abort the entire query, and either let the user resubmit the query or automatically run it again

As queries normally run for a few seconds or a few minutes at most, this way of handling errors is acceptable, since the cost of retrying is not too great. MPP databases also prefer to keep as much data as possible in memory

On the other hand, MapReduce can tolerate the failure of a map or reduce task without it affecting the job as a whole by retrying work at the granularity of an individual task. It is also very eager to write data to disk, partly for fault tolerance, and partly on the assumption that the dataset will be too big to fit in memory anyway.

In that case, rerunning the entire job due to a single task failure would be wasteful. Even if recovery at the granularity of an individual task introduces overheads that make fault-free processing slower

And this is why MapReduce is designed to tolerate frequent unexpected task termination: it’s not because the hardware is particularly unreliable, it’s because the freedom to arbitrarily terminate processes enables better resource utilization in a computing cluster.

# Beyond MapReduce

## Materialization of Intermediate State

. Publishing data to a well-known location in the distributed filesystem allows loose coupling so that jobs don’t need to know who is producing their input or consuming their output

the files on the distributed filesystem are simply intermediate state: a means of passing data from one job to the next.

The process of writing out this intermediate state to files is called materialization.

downside compared to unix
- only start when all tasks in the preceding jobs (that generate its inputs) have completed. Unix pipe are started at the same time, with output being consumed as soon as it is produced.
- Mappers are often redundant: they just read back the same file that was just written by a reducer, and prepare it for the next stage of partitioning and sorting. if the reducer output was partitioned and sorted in the same way as mapper output, then reducers could be chained together directly, without interleaving with mapper stages.
- Storing intermediate state in a distributed filesystem means those files are replicated across several nodes, which is often overkill for such temporary data.


# Dataflow engines

they handle an entire workflow as one job, rather than breaking it up into independent subjobs.

Since they explicitly model the flow of data through several processing stages, these systems are known as dataflow engines.

like MapReduce, they repeatedly calling a user-defined function to process one record at a time on a single thread.

Unlike in MapReduce, these functions need not take the strict roles of alternating map and reduce, but instead can be assembled in more flexible ways. We call these functions operators

connecting one operator’s output to another’s input:
- One option is to repartition and sort records by key
- Another possibility is to take several inputs and to partition them in the same way, but skip the sorting. This saves effort on partitioned hash joins, where the partitioning of records is important but the order is irrelevant
- For broadcast hash joins, the same output from one operator can be sent to all partitions of the join operator.

advantage:
- Expensive work such as sorting need only be performed in places where it is actually required, rather than always happening by default
- no unnecessary map tasks, since the work done by a mapper can often be incorporated into the preceding reduce operator
- scheduler has an overview of what data is required where, so it can make locality optimizations
- It is usually sufficient for intermediate state between operators to be kept in memory or written to local disk, which requires less I/O than writing it to HDFS(where it must be replicated to several machines and written to disk on each replica)
- Operators can start executing as soon as their input is ready; there is no need to wait for the entire preceding stage to finish before the next one starts.

significantly faster due to the optimizations

### Fault tolerance

An advantage of fully materializing intermediate state to a distributed filesystem is that it is durable, which makes fault tolerance fairly easy in MapReduce: if a task fails, it can just be restarted on another machine and read the same input again from the filesystem.

avoid writing intermediate state to HDFS, so they take a different approach to tolerating faults:
if a machine fails and the intermediate state on that machine is lost, it is recomputed from other data that is still available (a prior intermediary stage if possible, or otherwise the original input data, which is normally on HDFS).

To enable this recomputation, the framework must keep track of how a given piece of data was computed—which input partitions it used, and which operators were applied to it.

whether the computation is deterministic: that is, given the same input data, do the operators always produce the same output? This question matters if some of the lost data has already been sent to downstream operators.
The solution in the case of nondeterministic operators is normally to kill the downstream operators as well, and run them again on the new data.

In order to avoid such cascading faults, it is better to make operators deterministic.
hash table iteration. Such causes of nondeterminism need to be removed in order to reliably recover from faults, for example by generating pseudorandom numbers using a fixed seed.

### Discussion of materialization

Any operator that requires sorting will thus need to accumulate state, at least temporarily.

when using a dataflow engine, materialized datasets on HDFS are still usually the inputs and the final outputs of a job. The improvement over MapReduce is that you save yourself writing all the intermediate state to the filesystem as well.

## Graphs and Iterative Processing

using graphs for modeling data

the goal is to perform some kind of offline processing or analysis on an entire graph.

For example, one of the most famous graph analysis algorithms is PageRank [69], which tries to estimate the popularity of a web page based on what other web pages link to it. It is used as part of the formula that determines the order in which web search engines present their results.

Many graph algorithms are expressed by traversing one edge at a time, joining one vertex with an adjacent vertex in order to propagate some information, and repeating until some condition is met

This kind of algorithm is thus often implemented in an iterative style:
1. An external scheduler runs a batch process to calculate one step of the algorithm.

2. When the batch process completes, the scheduler checks whether it has finished (based on the completion condition—e.g., there are no more edges to follow, or the change compared to the last iteration is below some threshold).

3. If it has not yet finished, the scheduler goes back to step 1 and runs another round of the batch process.

### The Pregel processing model

As an optimization for batch processing graphs, the bulk synchronous parallel (BSP) model of computation [70] has become popular. It is also known as the Pregel model

one vertex can “send a message” to another vertex, and typically those messages are sent along the edges in a graph.

In each iteration, a function is called for each vertex, passing it all the messages that were sent to it—much like a call to the reducer.

The difference from MapReduce is that in the Pregel model, a vertex remembers its state in memory from one iteration to the next, so the function only needs to process new incoming messages.

### Fault tolerance

helps improve the performance of Pregel jobs, since messages can be batched and there is less waiting for communication. The only waiting is between iterations: since the Pregel model guarantees that all messages sent in one iteration are delivered in the next iteration, the prior iteration must completely finish, and all of its messages must be copied over the network, before the next one can start.

Pregel implementations guarantee that messages are processed exactly once at their destination vertex in the following iteration. Like MapReduce, the framework transparently recovers from faults in order to simplify the programming model for algorithms on top of Pregel.

This fault tolerance is achieved by periodically checkpointing the state of all vertices at the end of an iteration—i.e., writing their full state to durable storage. If a node fails and its in-memory state is lost, the simplest solution is to roll back the entire graph computation to the last checkpoint and restart the computation.

If the algorithm is deterministic and messages are logged, it is also possible to selectively recover only the partition that was lost

### Parallel execution

It is up to the framework to partition the graph—i.e., to decide which vertex runs on which machine, and how to route messages over the network so that they end up in the right place.

Because the programming model deals with just one vertex at a time (sometimes called “thinking like a vertex”), the framework may partition the graph in arbitrary ways.
vertices are colocated on the same machine if they need to communicate a lot. However, finding such an optimized partitioning is hard—in practice, the graph is often simply partitioned by an arbitrarily assigned vertex ID, making no attempt to group related vertices together.

As a result, graph algorithms often have a lot of cross-machine communication overhead, and the intermediate state (messages sent between nodes) is often bigger than the original graph. The overhead of sending messages over the network can significantly slow down distributed graph algorithms.

For this reason, if your graph can fit in memory on a single computer, it’s quite likely that a single-machine (maybe even single-threaded) algorithm will outperform a distributed batch process. 

If the graph is too big to fit on a single machine, a distributed approach such as Pregel is unavoidable; efficiently parallelizing graph algorithms is an area of ongoing research

## High-Level APIs and Languages

By now, the infrastructure has become robust enough

attention has turned to other areas: improving the programming model, improving the efficiency of processing, and broadening the set of problems that these technologies can solve.

As discussed previously, higher-level languages and APIs such as Hive, Pig, Cascading, and Crunch became popular because programming MapReduce jobs by hand is quite laborious.

These dataflow APIs generally use relational-style building blocks to express a computation: joining datasets on the value of some field; grouping tuples by key; filtering by some condition; and aggregating tuples by counting, summing, or other functions. Internally, these operations are implemented using the various join and grouping algorithms that we discussed earlier in this chapter.

Besides the obvious advantage of requiring less code, these high-level interfaces also allow interactive use, in which you write analysis code incrementally in a shell and run it frequently to observe what it is doing. This style of development is very helpful when exploring a dataset and experimenting with approaches for processing it. It is also reminiscent of the Unix philosophy

### The move toward declarative query languages

framework can analyze the properties of the join inputs and automatically decide which of the aforementioned join algorithms would be most suitable for the task at hand.

Hive, Spark, and Flink have cost-based query optimizers that can do this, and even change the order of joins so that the amount of intermediate state is minimized

However, dataflow engines have found that there are also advantages to incorporating more declarative features in areas besides joins.

if a callback function contains only a simple filtering condition, take advantage of column-oriented storage layouts, read only the required columns from disk.

### Specialization for different domains

standard processing patterns keep reoccurring, and so it is worth having reusable implementations of the common building blocks.

MPP databases have served the needs of business intelligence analysts and business reporting

Another domain of increasing importance is statistical and numerical algorithms, which are needed for machine learning applications such as classification and recommendation systems. Reusable implementations are emerging: for example, Mahout implements various algorithms for machine learning on top of MapReduce, Spark, and Flink, while MADlib implements similar functionality inside a relational MPP database



# Chapter 11 Stream Processing

stream processing: 
- a lot of data is unbounded because it arrives gradually over time
- so the dataset is never “complete” in any meaningful way
- simply processing every event as it happens. That is the idea behind stream processing.
- “stream” refers to data that is incrementally made available over time
- event streams as a data management mechanism: the unbounded, incrementally processed

what's covered:
- how streams are represented, stored, and transmitted over a network
- relationship between streams and databases
- approaches and tools for processing those streams continually, and ways that they can be used to build applications

## Transmitting Event Streams

a record is more commonly known as an event, but it is essentially the same thing: a small, self-contained, immutable object containing the details of something that happened at some point in time.

An event usually contains a timestamp indicating when it happened according to a time-of-day clock

An event may be encoded as a text string, or JSON, or perhaps in some binary form. This encoding allows you to store an event. It also allows you to send the event over the network

- an event is generated once by a producer (also known as a publisher or sender)
- potentially processed by multiple consumers (subscribers or recipients)
- related events are usually grouped together into a topic or stream

- producer writes every event that it generates to the datastore
- consumers to be notified when new events appear

Databases have traditionally not supported this kind of notification mechanism very well: relational databases commonly have triggers, which can react to a change. , but they are very limited in what they can do

### Messaging Systems

messaging system: a producer sends a message containing the event, which is then pushed to consumers

A direct communication channel like a Unix pipe or TCP connection between producer and consumer would be a simple way

most messaging systems expand on this basic model. In particular, Unix pipes and TCP connect exactly one sender with one recipient, whereas a messaging system allows multiple producer nodes to send messages to the same topic and allows multiple consumer nodes to receive messages in a topic.

publish/subscribe model, 
1. What happens if the producers send messages faster than the consumers can process them? 
    1. drop messages, 
    2. buffer messages in a queue - what happens as that queue grows? crash / write messages to disk (how does the disk access affect the performance)
    3. apply backpressure (also known as flow control; i.e., blocking the producer from sending more messages) - Unix pipes and TCP use backpressure because they have a small fixed-size buffer
2. What happens if nodes crash or temporarily go offline—are any messages lost?
    1. durability - require writing to disk / replication - with a cost
    2. If you can afford to sometimes lose messages, you can probably get higher throughput and lower latency

whether message lost is acceptable?
1. sensor readings and metrics that are transmitted periodically, an occasional missing data point is perhaps not important, since an updated value will be sent a short time later anyway
2. If you are counting events, it is more important that they are delivered reliably, since every lost message means incorrect counters.

### Direct messaging from producers to consumers

direct network communication between producers and consumers without going via intermediary nodes:
- UDP multicast
    - stock market feeds, where low latency is important
    - Although UDP itself is unreliable, application-level protocols can recover lost packets (producer must remember packets it has sent)
- Brokerless messaging libraries
    implementing publish/subscribe messaging over TCP or IP multicast
- unreliable UDP messaging for collecting metrics from all machines on the network and monitoring them
- If the consumer exposes a service on the network, producers can make a direct HTTP or RPC request to push messages to the consumer. (This is the idea behind webhooks [12], a pattern in which a callback URL of one service is registered with another service, and it makes a request to that URL whenever an event occurs.)

they generally require the application code to be aware of the possibility of message loss. The faults they can tolerate are quite limited: even if the protocols detect and retransmit packets that are lost in the network, they generally assume that producers and consumers are constantly online.

If a consumer is offline, it may miss messages that were sent while it is unreachable.
some retry failed message deliveries, but this approach may break down if the producer crashes, losing the buffer of messages that it was supposed to retry.

### Message brokers (aka. message queue)

a kind of database that is optimized for handling message streams
- runs as a server, with producers and consumers connecting to it as clients. 
- Producers write messages to the broker
- consumers receive them by reading them from the broker.

benefit
- By centralizing the data in the broker, these systems can more easily tolerate clients that come and go
- question of durability is moved to the broker instead (e.g. write data to disk so that they are not lost in case of a broker crash)

A consequence of queueing is also that consumers are generally asynchronous:
producer only waits for the broker to confirm that it has buffered the message and does not wait for the message to be processed by consumers.

### Message brokers compared to databases

- Databases usually keep data until it is explicitly deleted, 
    whereas most message brokers automatically delete a message when it has been successfully delivered to its consumers. Such message brokers are not suitable for long-term data storage.
- most message brokers assume that their working set is fairly small—i.e., the queues are short. If the broker needs to buffer a lot of messages because the consumers are slow (perhaps spilling messages to disk if they no longer fit in memory), each individual message takes longer to process, and the overall throughput may degrade
- Databases often support secondary indexes and various ways of searching for data, 
    while message brokers often support some way of subscribing to a subset of topics matching some pattern.
- When querying a database, the result is typically based on a point-in-time snapshot of the data; if another client subsequently writes something to the database that changes the query result, the first client does not find out that its prior result is now outdated.
    By contrast, message brokers do not support arbitrary queries, but they do notify clients when data changes (i.e., when new messages become available)

### Multiple consumers

When multiple consumers read messages in the same topic, two main patterns of messaging are used:
- Load balancing
    Each message is delivered to one of the consumers, so the consumers can share the work of processing the messages in the topic. The broker may assign messages to consumers arbitrarily. This pattern is useful when the messages are expensive to process, and so you want to be able to add consumers to parallelize the processing.
- Fan-out
    Each message is delivered to all of the consumers. Fan-out allows several independent consumers to each “tune in” to the same broadcast of messages, without affecting each other
The two patterns can be combined: for example, two separate groups of consumers may each subscribe to a topic, such that each group collectively receives all messages, but within each group only one of the nodes receives each message.

### Acknowledgments and redelivery

acknowledgments: a client must explicitly tell the broker when it has finished processing a message so that the broker can remove it from the queue.

If the connection to a client is closed or times out without the broker receiving an acknowledgment, it assumes that the message was not processed, and therefore it delivers the message again to another consumer. (it could happen that the message actually was fully processed, but the acknowledgment was lost)

When combined with load balancing, this redelivery behavior has an interesting effect on the ordering of messages.

the combination of load balancing with redelivery inevitably leads to messages being reordered. Message reordering is not a problem if messages are completely independent of each other, but it can be important if there are causal dependencies between messages

## Partitioned Logs

Sending a packet over a network or making a request to a network service is normally a transient operation that leaves no permanent trace.
vs
Databases and filesystems normally expected to be permanently recorded

This difference in mindset has a big impact on how derived data is created.

AMQP/JMS-style messaging: receiving a message is destructive if the acknowledgment causes it to be deleted from the broker. If you add a new consumer to a messaging system, it typically only starts receiving messages sent after the time it was registered; any prior messages are already gone and cannot be recovered.

log-based message brokers
- hybrid, combining the durable storage approach of databases with the low-latency notification facilities of messaging
- A log is simply an append-only sequence of records on disk
- a producer sends a message by appending it to the end of the log, and a consumer receives messages by reading the log sequentially.
- If a consumer reaches the end of the log, it waits for a notification that a new message has been appended.

In order to scale to higher throughput: 
- the log can be partitioned
- Different partitions can then be hosted on different machines
- separate log that can be read and written independently from other partitions
- A topic can then be defined as a group of partitions that all carry messages of the same type
- Within each partition, the broker assigns a monotonically increasing sequence number, or offset, to every message
- order: Such a sequence number makes sense because a partition is append-only, so the messages within a partition are totally ordered. There is no ordering guarantee across different partitions.

### Logs compared to traditional messaging

fan-out: 
The log-based approach trivially supports fan-out messaging, because several consumers can independently read the log without affecting each other

load-balancing:
To achieve load balancing across a group of consumers, instead of assigning individual messages to consumer clients, the broker can assign entire partitions to nodes in the consumer group.
downside:
- The number of nodes sharing the work of consuming a topic can be at most the number of log partitions in that topic
- If a single message is slow to process, it holds up the processing of subsequent messages in that partition

in situations where messages may be expensive to process and you want to parallelize processing on a message-by-message basis, and where message ordering is not so important, the JMS/AMQP style of message broker is preferable.

in situations with high message throughput, where each message is fast to process and where message ordering is important, the log-based approach works very well.

### Consumer offsets

Consuming a partition sequentially makes it easy to tell which messages have been processed: all messages with an offset less than a consumer’s current offset have already been processed

Thus, the broker does not need to track acknowledgments for every single messageit only needs to periodically record the consumer offsets. The reduced bookkeeping overhead and the opportunities for batching and pipelining in this approach help increase the throughput of log-based systems.

This offset is in fact very similar to the log sequence number that is commonly found in single-leader database replication. allows a follower to reconnect to a leader after it has become disconnected, and resume replication
- If a consumer node fails, another node in the consumer group is assigned the failed consumer’s partitions, and it starts consuming messages at the last recorded offset.
- If the consumer had processed subsequent messages but not yet recorded their offset, those messages will be processed a second time upon restart. We will discuss ways of dealing with this issue later in the chapter.

### Disk space usage

the log is actually divided into segments, and from time to time old segments are deleted or moved to archive storage.

a slow consumer cannot keep up with the rate of messages, and it falls so far behind that its consumer offset points to a deleted segment, it will miss some of the messages. Effectively, the log implements a bounded-size buffer that discards old messages when it gets full, also known as a circular buffer or ring buffer. However, since that buffer is on disk, it can be quite large.

In practice, deployments rarely use the full write bandwidth of the disk, so the log can typically keep a buffer of several days’ or even weeks’ worth of messages.

Regardless of how long you retain messages, the throughput of a log remains more or less constant, since every message is written to disk anyway [18]. This behavior is in contrast to messaging systems that keep messages in memory by default and only write them to disk if the queue grows too large:

such systems are fast when queues are short and become much slower when they start writing to disk, so the throughput depends on the amount of history retained.

### When consumers cannot keep up with producers

dropping messages, buffering, or applying backpressure. 

In this taxonomy, the log-based approach is a form of buffering with a large but fixed-size buffer (limited by the available disk space).

If a consumer falls so far behind that the messages it requires are older than what is retained on disk, it will not be able to read those messages. 
- As the buffer is large, there is enough time for a human operator to fix the slow consumer and allow it to catch up before it starts missing messages.
- Even if a consumer does fall too far behind and starts missing messages, only that consumer is affected; it does not disrupt the service for other consumers.
- This behavior also contrasts with traditional message brokers, where you need to be careful to delete any queues whose consumers have been shut down—otherwise they continue unnecessarily accumulating messages and taking away memory

### Replaying old messages

with AMQP- and JMS-style message brokers, processing and acknowledging messages is a destructive operation, since it causes the messages to be deleted on the broker.
vs
On the other hand, in a log-based message broker, consuming messages is more like reading from a file

The only side effect of processing, besides any output of the consumer, is that the consumer offset moves forward. But the offset is under the consumer’s control, so it can easily be manipulated if necessary

This aspect makes log-based messaging more like the batch processes of the last chapter, where derived data is clearly separated from input data through a repeatable transformation process. It allows more experimentation and easier recovery from errors and bugs, making it a good tool for integrating dataflows within an organization.

# Databases and Streams

log-based message brokers have been successful in taking ideas from databases and applying them to messaging. We can also go in reverse: take ideas from messaging and streams, and apply them to databases.

event is a record of something that happened at some point in time, it may also be a write to a database. 

a replication log is a stream of database write events, produced by the leader. The events in the replication log describe the data changes that occurred.

if every event represents a write to the database, and every replica processes the same events in the same order, then the replicas will all end up in the same final state.

we will first look at a problem that arises in heterogeneous data systems, and then explore how we can solve it by bringing ideas from event streams to databases.

### Keeping Systems in Sync

no single system that can satisfy all data storage, querying, and processing needs.

If periodic full database dumps are too slow, an alternative that is sometimes used is dual writes, in which the application code explicitly writes to each of the systems when data changes. 

concurrency:
race condition
Unless you have some additional concurrency detection mechanism

fault-tolerance:
Another problem with dual writes is that one of the writes may fail while the other succeeds. This is a fault-tolerance problem rather than a concurrency problem, but it also has the effect of the two systems becoming inconsistent with each other. Ensuring that they either both succeed or both fail is a case of the atomic commit problem, which is expensive to solve

The situation would be better if there really was only one leader—for example, the database—and if we could make the search index a follower of the database.

### Change Data Capture

The problem with most databases’ replication logs is that they have long been considered to be an internal implementation detail of the database, not a public API. Clients are supposed to query the database through its data model and query language, not parse the replication logs and try to extract data from them.

For decades, many databases simply did not have a documented way of getting the log of changes written to them. For this reason it was difficult to take all the changes made in a database and replicate them to a different storage technology

change data capture (CDC), which is the process of observing all data changes written to a database and extracting them in a form in which they can be replicated to other systems. CDC is especially interesting if changes are made available as a stream, immediately as they are written.

For example, you can capture the changes in a database and continually apply the same changes to a search index. If the log of changes is applied in the same order, you can expect the data in the search index to match the data in the database.

#### Implementing change data capture

We can call the log consumers derived data systems, b/c xxxxx is just another view onto the data in the system of record.

Change data capture is a mechanism for ensuring that all changes made to the system of record are also reflected in the derived data systems so that the derived systems have an accurate copy of the data.

Essentially, change data capture makes one database the leader, and turns the others into followers.

Database triggers can be used to implement change data capture. However, they tend to be fragile and have significant performance overheads. Parsing the replication log can be a more robust approach, although it also comes with challenges, such as handling schema changes.

Like message brokers, change data capture is usually asynchronous: the system of record database does not wait for the change to be applied to consumers before committing it. This design has the operational advantage that adding a slow consumer does not affect the system of record too much, but it has the downside that all the issues of replication lag apply (may see out-of-date data)

#### Initial snapshot

If you have the log of all changes that were ever made to a database, you can reconstruct the entire state of the database by replaying the log. However, in many cases, keeping all changes forever would require too much disk space, and replaying it would take too long, so the log needs to be truncated.

you need to start with a consistent snapshot. The snapshot of the database must correspond to a known position or offset in the change log, so that you know at which point to start applying changes after the snapshot

#### Log compaction

the storage engine periodically looks for log records with the same key, throws away any duplicates, and keeps only the most recent update for each key. This compaction and merging process runs in the background.

In a log-structured storage engine, an update with a special null value (a tombstone) indicates that a key was deleted, and causes it to be removed

The disk space required for such a compacted log depends only on the current contents of the database, not the number of writes that have ever occurred. previous values will eventually be garbagecollected, and only the latest value will be retained.

whenever you want to rebuild a derived data system such as a search index, you can start a new consumer from offset 0 of the log-compacted topic, and sequentially scan over all messages in the log. The log is guaranteed to contain the most recent value for every key in the database —in other words, you can use it to obtain a full copy of the database contents without having to take another snapshot of the CDC source database.

#### API support for change streams

Increasingly, databases are beginning to support change streams as a first-class interface, rather than the typical retrofitted and reverse-engineered CDC efforts. For example, RethinkDB allows queries to subscribe to notifications when the results of a query change

subscribe to data changes and update the user interface

The database represents an output stream in the relational data model as a table into which transactions can insert tuples, but which cannot be queried. The stream then consists of the log of tuples that committed transactions have written to this special table, External consumers can asynchronously consume this log and use it to update derived data systems.

## Event Sourcing

event sourcing, a technique that was developed in the domain-driven design (DDD) community [42, 43, 44]. We will discuss event sourcing briefly, because it incorporates some useful and relevant ideas for streaming systems.

event sourcing involves storing all changes to the application state as a log of change events. The biggest difference is that event sourcing applies the idea at a different level of abstraction:

- In change data capture, the application uses the database in a mutable way, updating and deleting records at will. 
    - The log of changes is extracted from the database at a low level (e.g., by parsing the replication log), which ensures that the order of writes extracted from the database matches the order in which they were actually written, avoiding the race condition
    - The application writing to the database does not need to be aware that CDC is occurring
- In event sourcing, the application logic is explicitly built on the basis of immutable events that are written to an event log.
    - the event store is appendonly, and updates or deletes are discouraged or prohibited
    - Events are designed to reflect things that happened at the application level, rather than low-level state changes.
Event sourcing is a powerful technique for data modeling
- from an application point of view it is more meaningful to record the user’s actions as immutable events, rather than recording the effect of those actions on a mutable database.
    Event sourcing makes it easier to evolve applications over time, helps with debugging by making it easier to understand after the fact why something happened, and guards against application bugs
    expresses the intention in a neutral fashion

### Deriving current state from the event log

An event log by itself is not very useful, because users generally expect to see the current state of a system, not the history of modifications.

Thus, applications that use event sourcing need to take the log of events (representing the data written to the system) and transform it into application state that is suitable for showing to a user. 
- it should be deterministic so that you can run it again and derive the same application state

Like with change data capture, replaying the event log allows you to reconstruct the current state of the system. However, log compaction needs to be handled differently:
- A CDC event for the update of a record typically contains the entire new version of the record
    so log compaction can discard previous events for the same key.
- with event sourcing, events are modeled at a higher level
    an event typically expresses the intent of a user action, not the mechanics of the state update that occurred as a result of the action.
    later events typically do not override prior events, and so you need the full history of events to reconstruct the final state. Log compaction is not possible in the same way.

Applications that use event sourcing typically have some mechanism for storing snapshots of the current state that is derived from the log of events, so they don’t need to repeatedly reprocess the full log

### Commands and events

The event sourcing philosophy is careful to distinguish between events and commands
- When a request from a user first arrives, it is initially a command: at this point it may still fail, for example because some integrity condition is violated.
- The application must first validate that it can execute the command
- If the validation is successful and the command is accepted, it becomes an event, which is durable and immutable.
- At the point when the event is generated, it becomes a fact
- A consumer of the event stream is not allowed to reject an event, it is already an immutable part of the log. Thus, any validation of a command needs to happen synchronously, before it becomes an event

Alternatively, the user request to reserve a seat could be split into two events: first a tentative reservation, and then a separate confirmation event once the reservation has been validated. This split allows the validation to take place in an asynchronous process.

### State, Streams, and Immutability

This principle of immutability is also what makes event sourcing and change data capture so powerful.

We normally think of databases as storing the current state of the application—this representation is optimized for reads, and it is usually the most convenient for serving queries. The nature of state is that it changes, so databases support updating and deleting data as well as inserting it.

- Whenever you have state that changes, that state is the result of the events that mutated it over time.
- No matter how the state changes, there was always a sequence of events that caused those changes.
- The key idea is that mutable state and an append-only log of immutable events do not contradict each other
- The log of all changes, the changelog, represents the evolution of state over time.

If you are mathematically inclined, you might say that the application state is what you get when you integrate an event stream over time, and a change stream is what you get when you differentiate the state by time.

If you store the changelog durably, that simply has the effect of making the state reproducible.

High-speed appends are the only way to change the log. From this perspective, the contents of the database hold a caching of the latest record values in the logs. The truth is the log. The database is a cache of a subset of the log. That cached subset happens to be the latest value of each record and index value from the log.

Log compaction, as discussed in “Log compaction” on page 456, is one way of bridging the distinction between log and database state: it retains only the latest version of each record, and discards overwritten versions.

#### Advantages of immutable events

if you accidentally deploy buggy code that writes bad data to a database, recovery is much harder if the code is able to destructively overwrite data. With an append-only log of immutable events, it is much easier to diagnose what happened and recover from the problem.

Immutable events also capture more information than just the current state.

e.g. a customer may add an item to their cart and then remove it again. 
- This information is recorded in an event log, but would be lost in a database that deletes items when they are removed from the cart

#### Deriving several views from the same event log

Moreover, by separating mutable state from the immutable event log, you can derive several different read-oriented representations from the same log of events.

This works just like having multiple consumers of a stream

Having an explicit translation step from an event log to a database makes it easier to evolve your application over time: 
- you can use the event log to build a separate read-optimized view for the new feature, and run it alongside the existing systems. easier than performing a complicated schema migration


Storing data is normally quite straightforward if you don’t have to worry about how it is going to be queried and accessed; many of the complexities of schema design, indexing, and storage engines are the result of wanting to support certain query and access patterns

you gain a lot of flexibility by separating the form in which data is written from the form it is read, and by allowing several different read views. This idea is sometimes known as command query responsibility segregation (CQRS)

The traditional approach to database and schema design is based on the fallacy that data must be written in the same form as it will be queried.

Debates about normalization and denormalization become largely irrelevant if you can translate data from a write-optimized event log to read-optimized application state: 
it is entirely reasonable to denormalize data in the read-optimized views, as the translation process gives you a mechanism for keeping it consistent with the event log.

read-optimized state: home timelines are highly denormalized, since your tweets are duplicated in all of the timelines of the people following you. However, the fan-out service keeps this duplicated state in sync with new tweets and new following relationships, which keeps the duplication manageable.

#### Concurrency control

The biggest downside of event sourcing and change data capture is that the consumers of the event log are usually asynchronous, so there is a possibility that a user may make a write to the log, then read from a log-derived view and find that their write has not yet been reflected

One solution would be to perform the updates of the read view synchronously with appending the event to the log. 
This requires a transaction to combine the writes into an atomic unit, 
- so either you need to keep the event log and the read view in the same storage system, 
- or you need a distributed transaction across the different systems.

deriving the current state from an event log also simplifies some aspects of concurrency control

Much of the need for multi-object transaction action requiring data to be changed in several different places. 
With event sourcing, you can design an event such that it is a self-contained description of a user action. 
The user action then requires only a single write in one place—namely appending the events to the log—which is easy to make atomic.

If the event log and the application state are partitioned in the same way, then a straightforward single-threaded log consumer needs no concurrency control for writes. The log removes the nondeterminism of concurrency by defining a serial order of events in a partition

#### Limitations of immutability

Many systems that don’t use an event-sourced model nevertheless rely on immutability: various databases internally use immutable data structures or multi-version data to support point-in-time snapshots. Version control systems such as Git, Mercurial, and Fossil also rely on immutable data to preserve version history of files.

To what extent is it feasible to keep an immutable history of all changes forever?
- The answer depends on the amount of churn in the dataset.
    - Some workloads mostly add data and rarely update or delete; they are easy to make immutable.
    - Other workloads have a high rate of updates and deletes on a comparatively small dataset
        - the immutable history may grow prohibitively large, 
        - fragmentation may become an issue, 
        - and the performance of compaction and garbage collection becomes crucial for operational robustness
    
you need data to be deleted for administrative reasons, in spite of all immutability. 
- privacy regulations may require deleting a user’s personal information after they close their account, 
- data protection legislation may require erroneous information to be removed
- you actually want to rewrite history and pretend that the data was never written

Truly deleting data is surprisingly hard [64], since copies can live in many places
- for example, storage engines, filesystems, and SSDs often write to a new location rather than overwriting in place [52], 
- and backups are often deliberately immutable to prevent accidental deletion or corruption. 
- Deletion is more a matter of “making it harder to retrieve the data” than actually “making it impossible to retrieve the data.”

## Processing Streams

what you can do with the stream once you have it
1. write it to a database, cache, search index, or similar storage system, from where it can then be queried by other clients. keeping a database in sync with changes happening in other parts of the system
2. You can push the events to users in some way, for example by sending email alerts or push notifications, or by streaming the events to a real-time dashboard where they are visualized. In this case, a human is the ultimate consumer of the stream.
3. **You can process one or more input streams to produce one or more output streams.**

operator or a job: A piece of code that processes streams

pattern of dataflow is similar: a stream processor consumes input streams in a read-only fashion and writes its output to a different location in an append-only fashion.

The patterns for partitioning and parallelization in stream processors are also very similar to those in MapReduce and the dataflow engines

The one crucial difference to batch jobs is that a stream never ends.
- so sort-merge joins cannot be used.
- Fault-tolerance mechanisms must also change - restarting from the beginning after a crash may not be a viable option.

### Uses of Stream Processing

Stream processing has long been used for monitoring purposes, where an organization wants to be alerted if certain things happen.
- Fraud detection systems need to determine if the usage patterns of a credit card have unexpectedly changed, and block the card if it is likely to have been stolen.
- Trading systems need to examine price changes in a financial market and execute trades according to specified rules.
- Manufacturing systems need to monitor the status of machines in a factory, and quickly identify the problem if there is a malfunction.
- Military and intelligence systems need to track the activities of a potential aggressor, and raise the alarm if there are signs of an attack.

#### Complex event processing

Complex event processing (CEP) is an approach developed in the 1990s for analyzing event streams, especially geared toward the kind of application that requires searching for certain event patterns. 

CEP, which allows searching for patterns consisting of multiple events,

CEP allows you to specify rules to search for certain patterns of events in a stream.

CEP systems often use a high-level declarative query language like SQL, or a graphical user interface, to describe the patterns of events that should be detected.

These queries are submitted to a processing engine that consumes the input streams and internally maintains a state machine that performs the required matching.

When a match is found, the engine emits a complex event (hence the name) with the details of the event pattern that was detected

In these systems, the relationship between queries and data is reversed compared to normal databases. 
- Usually, a database stores data persistently and treats queries as transient
- CEP engines reverse these roles: queries are stored long-term, and events from the input streams continuously flow past them

#### Stream analytics

Another area in which stream processing is used is for analytics on streams. The boundary between CEP and stream analytics is blurry, but as a general rule, analytics tends to be less interested in finding specific event sequences and is more oriented toward aggregations and statistical metrics over a large number of events

- Measuring the rate of some type of event (how often it occurs per time interval)
- Calculating the rolling average of a value over some time period
- Comparing current statistics to previous time intervals (e.g., to detect trends or to alert on metrics that are unusually high or low compared to the same time last week)

Such statistics are usually computed over fixed time intervals

window: The time interval over which you aggregate

Probabilistic algorithms produce approximate results, but have the advantage of requiring significantly less memory in the stream processor than exact algorithms. probabilistic algorithms are merely an optimization

#### Maintaining materialized views

materialized views: deriving an alternative view onto some dataset so that you can query it efficiently, and updating that view whenever the underlying data changes

#### Search on streams

search for individual events based on complex criteria, such as full-text search queries.

e.g. media monitoring services subscribe to feeds of news articles and broadcasts from media outlets, and search for any news mentioning companies, products, or topics of interest.

formulating a search query in advance, and then continually matching the stream of news items against this query.

be notified when a new property matching their search criteria appears on the market.

To optimize the process, it is possible to index the queries as well as the documents, and thus narrow down the set of queries that may match

#### Message passing and RPC

### Reasoning About Time

batch process:
There is no point in looking at the system clock of the machine running the batch process, because the time at which the process is run has nothing to do with the time at which the events actually occurred.

stream processing frameworks:
use the local system clock on the processing machine (the processing time) to determine windowing [79]. This approach has the advantage of being simple, and it is reasonable if the delay between event creation and event processing is negligibly short. However, it breaks down if there is any significant processing lag

#### Event time versus processing time

There are many reasons why processing may be delayed: queueing, network faults, a performance issue leading to contention in the message broker or processor, a restart of the stream consumer, or reprocessing of past events.

message delays can also lead to unpredictable ordering of messages.

Confusing event time and processing time leads to bad data.

#### Knowing when you’re ready

A tricky problem when defining windows in terms of event time is that you can never be sure when you have received all of the events for a particular window

straggler events: arrive after the window has already been declared complete
1. Ignore the straggler events (You can track the number of dropped events as a metric, and alert if you start dropping a significant amount of data.)
2. Publish a correction. an updated value for the window with stragglers included. You may also need to retract the previous output.

#### Whose clock are you using, anyway?

Assigning timestamps to events is even more difficult when events can be buffered at several points in the system.

The time at which the event was received by the server (according to the server’s clock) is more likely to be accurate, since the server is under your control, but less meaningful in terms of describing the user interaction.

log three timestamps [82]:

• The time at which the event occurred, according to the device clock

• The time at which the event was sent to the server, according to the device clock

• The time at which the event was received by the server, according to the server clock

By subtracting the second timestamp from the third, you can estimate the offset between the device clock and the server clock (assuming the network delay is negligible compared to the required timestamp accuracy). You can then apply that offset to the event timestamp, and thus estimate the true time at which the event actually occurred (assuming the device clock offset did not change between

#### Types of windows

Tumbling window
    fixed length, and every event belongs to exactly one window.
Hopping window
    fixed length, but allows windows to overlap in order to provide some smoothing. hop size describe overlap
Sliding window
    contains all the events that occur within some interval of each other. keeping a buffer of events sorted by time and removing old events when they expire from the window
Session window
    a session window has no fixed duration. Instead, it is defined by grouping together all events for the same user that occur closely together in time, and the window ends when the user has been inactive for some time

### Stream Joins

#### stream-stream joins

Say you have a search feature on your website, and you want to detect recent trends in searched-for URLs. Every time someone types a search query, you log an event containing the query and the results returned. Every time someone clicks one of the search results, you log another event recording the click.

a stream processor needs to maintain state: for example, all the events that occurred in the last hour, indexed by session ID.

Whenever a search event or click event occurs, it is added to the appropriate index, and the stream processor also checks the other index to see if another event for the same session ID has already arrived. If there is a matching event, you emit an event saying which search result was clicked. If the search event expires without you seeing a matching click event, you emit an event saying which search results were not clicked.

#### stream-table joins

This process is sometimes known as enriching the activity events with information from the database.

load a copy of the database into the stream processor so that it can be queried locally without a network round-trip.

solved by change data capture: the stream processor can subscribe to a changelog of the user profile database

#### table-table joins (materialized view maintenance)

we want a timeline cache: a kind of per-user “inbox”
- When user u sends a new tweet, it is added to the timeline of every user who is following u.
- When a user deletes a tweet, it is removed from all users’ timelines.
- When user u 1 starts following user u2 , recent tweets by u 2 are added to u1 ’s timeline.
- When user u 1 unfollows user u2 , tweets by u 2 are removed from u1 ’s timeline.

you need streams of events for tweets (sending and deleting) and for follow relationships (following and unfollowing).

The stream process needs to maintain a database containing the set of followers for each user so that it knows which timelines need to be updated when a new tweet arrives

it maintains a materialized view for a query that joins two tables (tweets and follows)

#### Time-dependence of joins

they all require the stream processor to maintain some state (search and click events, user profiles, or follower list) based on one join input, and query that state on messages from the other join input.

The order of the events that maintain the state is important (it matters whether you first follow and then unfollow, or the other way round).

if state changes over time, and you join with some state, what point in time do you use for the join

- e.g. apply the right tax rate to invoices - date of sale

If the ordering of events across streams is undetermined, the join becomes nondeterministic

In data warehouses, this issue is known as a slowly changing dimension (SCD), and it is often addressed by using a unique identifier for a particular version of the joined record

for example, every time the tax rate changes, it is given a new identifier, and the invoice includes the identifier for the tax rate at the time of sale [88, 89]. This change makes the join deterministic, but has the consequence that log compaction is not possible, since all versions of the records in the table need to be retained.

### Fault Tolerance

#### Microbatching and checkpointing

microbatching: One solution is to break the stream into small blocks, and treat each block like a miniature batch process.

smaller batches incur greater scheduling and coordination overhead, while larger batches mean a longer delay before results of the stream processor become visible.

Microbatching also implicitly provides a tumbling window equal to the batch size (windowed by processing time, not event timestamps)

periodically generate rolling checkpoints of state and write them to durable storage

If a stream operator crashes, it can restart from its most recent checkpoint and discard any output generated between the last checkpoint and the crash. The checkpoints are triggered by barriers in the message stream, similar to the boundaries between microbatches, but without forcing a particular window size.

the microbatching and checkpointing approaches provide the same exactly-once semantics as batch processing. However, as soon as output leaves the stream processor, the microbatching and checkpointing approaches provide the same exactly-once semantics as batch processing. However, as soon as output leaves the stream processor, 

#### Atomic commit revisited

we need to ensure that all outputs and side effects of processing an event take effect if and only if the processing is successful. 
Those effects include 
- any messages sent to downstream operators or external messaging systems (including email or push notifications), 
- any database writes, 
- any changes to operator state, and 
- any acknowledgment of input messages

Those things either all need to happen atomically, or none of them must happen, but they should not go out of sync with each other.

However, in more restricted environments it is possible to implement such an atomic commit facility efficiently

Unlike XA, these implementations do not attempt to provide transactions across heterogeneous technologies, but instead keep them internal by managing both state changes and messaging within the stream processing framework. The overhead of the transaction protocol can be amortized by processing several input messages within a single transaction.

#### Idempotence

Our goal is to discard the partial output of any failed tasks so that they can be safely retried without taking effect twice.

An idempotent operation is one that you can perform multiple times, and it has the same effect as if you performed it only once.

e.g. setting a key in a key-value store to some fixed value is idempotent

Even if an operation is not naturally idempotent, it can often be made idempotent with a bit of extra metadata.

every message has a persistent, monotonically increasing offset. When writing a value to an external database, you can include the offset of the message that triggered the last write with the value. Thus, you can tell whether an update has already been applied, and avoid performing the same update again.

When failing over from one processing node to another, fencing may be required (see “The leader and the lock” on page 301) to prevent interference from a node that is thought to be dead but is actually alive. Despite all those caveats, idempotent operations can be an effective way of achieving exactly-once semantics with only a small overhead.

#### Rebuilding state after a failure

Any stream process that requires state—for example, any windowed aggregations (such as counters, averages, and histograms) and any tables and indexes used for joins—must ensure that this state can be recovered after a failure.

One option is to keep the state in a remote datastore and replicate it

An alternative is to keep state local to the stream processor, and replicate it periodically. Then, when the stream processor is recovering from a failure, the new task can read the replicated state and resume processing without data loss.

In some cases, it may not even be necessary to replicate the state, because it can be rebuilt from the input streams.

if the state consists of aggregations over a fairly short window, it may be fast enough to simply replay the input events corresponding to that window. If the state is a local replica of a database, maintained by change data capture, the database can also be rebuilt from the log-compacted change stream

all of these trade-offs depend on the performance characteristics of the underlying infrastructure: 
- in some systems, network delay may be lower than disk access latency, and network bandwidth may be comparable to disk bandwidth. There is no universally ideal trade-off for all situations.
