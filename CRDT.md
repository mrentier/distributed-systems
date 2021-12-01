<h1>The sum of unique numbers</h1>

A person is emailing unique numbers to two email accounts. The owner of each receiving email account starts at zero and adds the value they receive to their current value. For example: 3 emails are sent out. The first email contains number 1, the second email contains number 3 and the last email contains number 6. Each receiver starts at zero and is expected to end at a value of 10 (0+1+3+6). However, emails can be received in a different order than they were sent, the sender might forget an addressee or emails can get lost or duplicated. Can the two receivers come to a consensus on the final sum? Absolutely! Each receiver can keep a collection with unique values it received (a set). As long as the receivers can communicate successfully, they can merge the values (set union) that each has received and come to a final sum. Since addition is communicative, the order in which emails were received does not impact the result. 

Note that the final consensus between the receivers might not match the expected value of 10. For example: when both receivers missed the email with number 6, the consensus will be that the final value is 4.

The set-based solution exists because the problem has some very specific properties:
- The order in which messages are delivered does not matter.
- We know the numbers are unique, so we can ignore duplicates.
- Emails cannot be deleted (retracted) or modified after they were sent.

<h1>The problem with distributed replicas</h1>

The above example of 'the sum of unique numbers' demonstrates a rather fundamental issue in distribute systems: how to reach consensus amongst several nodes when different events occur across the nodes. Broadly speaking, there are two possible ways of dealing with such differences: strongly consistent replication or optimistic replication.

<h2>Strongly consistent replication</h2>

In this model, the replicas (nodes) coordinate with each other to decide when and how to apply the modifications. This approach enables strong consistency models such as serializable transactions and linearizability. However, waiting for this coordination reduces the performance of these systems. Also, the CAP theorem tells us that it is impossible to make data changes on a replica while it is disconnected from the rest of the system (e.g. due to a network partition, or because it is a mobile device with intermittent connectivity). Thus strongly consistent replication tends to compromise on availability when facing consistency and network partition challenges, resulting in a CP system according to the CAP theorem.

 <h2>Optimistic replication</h2>

In this model, users may modify the data on any replica independently of any other replica, even if the replica is offline or disconnected from the others. This approach enables maximum performance and availability, but it can lead to conflicts when multiple clients or users concurrently modify the same piece of data. These conflicts then need to be resolved when the replicas communicate with each other. Optimistic replication tends to compromise on consistency when facing availability and network partition challenges, resulting in an AP system according to the CAP theorem.

<h1>Conflict-free Replicated Data Types</h1>

The 'sum of unique numbers' example utilizes a data-structure called a grow-only set. A grow-only set is one of a group of data types named Conflict-free Replicated Data Types (CRDTs). CRDTs fall in the optimistic replication model. Conflict-free is a vague description, but it comes down to the following: <em>we are operating on data structures that don't require exclusive write access and are 1) able to detect concurrent updates and 2) perform deterministic, automatic conflict resolution</em>. This doesn't imply a lack of conflict, but we are able to always determine the output up front, based on metadata contained within the structure itself. The core CRDTs are counters, registers and sets, but from them we can compose more advanced ones like maps or even JSON.

There are many implementations of CRDTs in use today and they have been instrumental in distributed computing at scale. Examples are:

- Amazon uses CRDTs to keep their order cart in sync. Their database, known as Dynamo, is available as a service.
- Riak is one of the most popular solutions in this area. One of their well-known customers are Riot games, which uses Riak to implement their in-game chat.
- Rovio (the company behind Angry Birds game series) uses conflict-free counters for their advertisement platform to make their impression counters work freely even in offline scenarios.
- SoundClound has their own implementation of Last-Write-Wins Set build in Go on top of Redis, known as Roshi, which they use for their observers management.
- TomTom makes use of CRDTs to manage their navigation data.
- CREAustralia uses them for their click stream analytics.  
- Mobile apps that store data on the local device, and that need to sync that data to other devices belonging to the same user (such as calendars, notes, contacts, or reminders);
- Collaboration software, such as Google Docs, Trello, Figma, or many others, in which several users can concurrently make changes to the same file or data. Note though that many collaboration tools use more operation-based solutions due to issues with intent (CRDTs replicate data but don't include by default a notion of intent such as the intent of removing a whole paragraph even when another user is editing that same paragraph).

<h2>Can I use an existing solution?</h2>

Distributed systems are complex. If you need a highly-available distributed data store it is likely your best option to pick an existing solution and move on. However, it is important to realize that there are many trade-offs to be made in the design of distributed systems and any of the existing solutions that you might pick will contain a series of these trade-offs. An understanding of the trade-offs will help you choose the correct solution for your problem.

<h1>CRDTs in practice</h1>
 
In practice a CRDT brings:

- local updates without needing remote synchronization.
- local merge upon receiving data from other nodes (communication with other nodes is assumed, i.e. gossip protocol)
- a guarantee that all local merges converge. THis is under the assumption that nodes will communicate at some point in time (liveness).

 Key to a CRDT is the merge operation. A merge operation is a binary operation (an operation on two CRDTs) and has the following properties:
- commutative: x.y = y.x
- associative: (x.y).z = x.(y.z)
- idempotent: x.x = x

A merge is required to be Idempotent because in distributed system events can be delivered more than once. Commutative and associative properties are required because in distributed systems the order of events is not guaranteed. The merge operation however requires the same result independent of the order in which events were received locally so that it can ensure that all local merges converge.

<h2>Grow-only counter</h2>

A simple example of a CRDT is a grow-only counter. Lets assume we have three nodes A, B and C with the following state (each node lists its state and the known state for the other nodes):
- node A: A:1 B:0 C:0
- node B: A:0 B:4 C:0
- node C: A:0 B:0 c:3

The merge is the maximum of the corresponding elements: A:1 B:4 C:3. The Max function is a valid merge function because it is commutative, associative and idempotent.The value of the counter is the sum of all elements (1+4+3=8).

<h2>Grow-only set</h2>

Assume that each node has a set and adds elements to its own set. The data type for the G-Set is an array of sets with each set belonging to a node.
- Node A: A:{x,y} B:{} C:{}
- Node B: A:{} B:{z} C:{}
- Node C: A:{} B:{} C:{a,b,c}

 The merge operation is the set union of all corresponding sets: A:{x,y} B:{z} C:{a,b,c}. Note that the union satisfies the commutative, associative and idempotent properties that are required for a merge operation. The total value is the union of all sets: {x,y,z,a,b,c}

 <h1>A holiday gift list</h1>

The complexity of CRDTs increases once you consider add, delete and modify operations on data. Quite often some additional form of ordering is needed in order to support delete and modify operations (i.e. causal relationships, 'last wins' or 'add wins'). Let's look at a more complex CRDT for synchronizing a holiday gift list. Folowing are some of our requirements for the holiday gift list application:

- A user can add, delete or modify gift suggestions at any time.
- Replicas of the gift list can be stored across multiple devices.
- Devices are not always available (can for example be turned off or not have a network connection).

 A naive solution involves clients sending their local version of the holiday gift list to a server. The server synchronizes the gift list by using a 'Last Write Wins' strategy. This can result in inconsistencies due to:

- the time when updates are sent to the server being different from the time an update was made.
- network latency.
- unreliable clocks on the server. Later on we will expand on why physical clocks can be problematic.

Why are CRDTs a good candidate for this problem?
- Changes are made instantly on the local node.
- Synchronization is decentralized.
- Order is not important.
- Latency, failed requests etc. have no impact.

 Lets review a number of set-based CRDTs and see which would be appropriate for our holiday gift list application.

<h2>2P-Set</h2>

A 2P-set maintains two sets: one set with additions and one set with removals (also known as a tombstone set). The merge operation is defined as the union between the two add-sets and the union of the two remove sets. The final value is defined by all elements in the add-set that are not in the remove set. 
An example of a 2P-set is the following:
- Add-Set: A:{"drill","ipad"} B:{"drill","coffee cup"}
- Remove Set: A:{"drill"} B:{}
- Merge Add-Set: {"drill","ipad","coffee cup"}
- Merge Remove-Set: {"drill"}
- Lookup: {"ipad","coffee cup""}

A 2P-set is however not a good solution to our problem, since: 
- a gift can only be removed once and never added again (once an item is in the tombstone set it is never removed from the tombstone set).
- an item can't be modified.

<h2>LWW-element set</h2>

A last-write-wins-element set (LWW-element set) attaches a timestamp to each element. A LWW-element set has the following properties:
- You can add again by adding the element with a higher timestamp than the one in the remove set.
- The merge operation is defined by taking the union of the add-sets and remove-sets.
- The lookup is defined by all elements in the add-set that are not also in remove-set with a higher timestamp.

Following is an example:
- Add-Set: A {(1,"drill"), (1,"ipad")} B: {5,"drill"),(1,"coffee cup")}
- Remove-Set: A:{(3,"drill")} B:{(1, "drill")}
- Merge Add-Set: {(1,"drill"),(5,"drill"),(1"ipad"),(1,"coffee cup")}
- Merge Remove-Set: {(1,"drill"),(3,"drill")}
- Lookup: {"drill","ipad","coffee cup"}

The LLW-element set does however not support modifying an element.
 
<h2>OUR-Set</h2>

The Observed-Update-Remove Set (OUR set) allows an identifier to remain the same while an element is changed. Each element has a timestamp which is updated on any change. Instead of a remove-set we utilize a single set and each item has a removed flag. For the merge operation we take the union of the two sets and for every element with the same identifier we take the one with the highest timestamp. The final value is determined by all elements that do not have the removed flag set.
As an example:
- Set A: { (#a,1,"drill",removed), (#b, 2, "ipad", removed)} B: {(#a,5,"socks"),(#c,1,"coffee cop"),(#b,1,"ipad")}
- Merge set: {(#a,1,"drill",removed), (#b,2,"ipad",removed), (#a,5,"socks"),(#c,1,"coffee cup")}
- Lookup: {"socks","coffee cup"}

<h2>Final thoughts about CRDTs</h2>

What about garbage? CRDTs tend to potentially grow unbounded because of tombstones (items deleted from our gift list). When can we prune deleted gift ideas? Each node holding a gift list set should have seen the deleted element before it can be pruned. Otherwise a deleted element could be resurrected. This means we should capture a timestamp of the last synchronization between a client and the service. Another option is only sending differences upon any update.

<h1>About clocks</h1>

The concept of timestamps was introduced without a second thought. Timestamps are however somewhat problematic in distributed systems. As Jonas Boner observed: "Tracking time is actually tracking causality" (Jonas Boner, "Life Beyond the Illusion of Present"). The term causality in distributed systems originates from a concept in physics where "causal connections gives us the only ordering of events that all observers will agree on". 

<h2>The problems with physical clocks</h2>

One approach for reasoning about causality relations in distributed systems is by using a physical clock. Physical clocks are however problematic due to: 
- hardware wonkiness pushing clocks days or centuries into the future or past.
- virtualization wreaking havoc on kernel timekeeping.
- misconfigured nodes not having NTP enabled, or may not be able to reach upstream sources.
- upstream NTP servers providing wrong information. When the problem is identified and fixed, NTP corrects large time differentials by jumping the clock discontinously to the correct time.
- POSIX time itself not being monotonic (leap seconds).
 
 In short: in general there is no physical clock for distributed systems.  A causal history (or compressed representation) is necessary to determine the partial ordering of events or detect inconsistent data replicas because physical clocks are unreliable.

<h2>Clocks that do not keep time</h2>

An alternative to a physical clock is a logical clock: a clock that keeps relationships rather than time. We will look at a specific kind of logical clock named a vector clock. 

A vector clock tracks the last event you have observed from other processes (nodes) that you communicate with and the last event you have sent. Although the received events might not influence the next event you send, the fact that you have received the events means that they <em>could</em> influence the next event you send. Any events that you have not received cannot influence your actions. It is important to note that a logical clock is specific to a node and is not global across all nodes. This means that at any point in time it is likely that nodes have different logical clocks.

A physical clock attempted to enforce a total order on all events. Given any two events across nodes, we could determine which event came first. Logical clocks only provide you a partial ordering of events. There are events across actors that we do not know the order of. For example when two nodes send events but do not receive events then we cannot determine the order of all events across all nodes.

<h2>Data consistency with version clocks</h2>

It turns out that vector clocks are very useful for deriving causality among events. But for data consistency we can get away with a more light-weight option called a version clock. The goal is to derive the final state of a piece of data in a node without having to be concerned about the exact sequence of events that caused the final state. For example: when one node has 10 changes pending and no other nodes have changes, it is only important to know that the node with changes holds the final state and that final state was determined by the last event. The node only has to attach the final version info when passing a chain of predecessors. Thus a version clock provides a summary of vector clocks. Although sometimes used interchangeably, vector clocks and version clocks have similar structure but a different meaning.

<h1>References</h1>

https://github.com/fallin/Itc4net

https://aphyr.com/posts/299-the-trouble-with-timestamps

https://www.tiny.cloud/blog/real-time-collaboration-ot-vs-crdt/

http://serverless.com//blog/crdt-explained-supercharge-serverless-at-edge

https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type

https://bartoszsypytkowski.com/the-state-of-a-state-based-crdts/

https://georgerpu.github.io/2020/07/20/crdt/

