
#  RAFT Consensus Algorithm: Leader Election Process

The **RAFT (Reliable, Available, Fault-tolerant) Consensus Algorithm** is designed to manage a replicated log in a distributed system, ensuring **consistency** and **fault tolerance**. Its primary goal is to be understandable, which it achieves by breaking the consensus problem into three subproblems: **Leader Election**, **Log Replication**, and **Safety**.

This guide focuses on the **Leader Election** process, which is responsible for selecting a single, consistent leader among a cluster of nodes.

---

## Core Concepts

The RAFT election process relies on the following key principles:

| Concept | Description |
| :--- | :--- |
| **Node States** | A node can be a **Follower** (passive), **Candidate** (seeking election), or **Leader** (managing log replication). |
| **Term Numbers** | A logical clock that increments with each election. A higher term always indicates a more up-to-date state. |
| **Election Timeout** | A randomized timer (typically 150-300ms) that prevents split-brain. If a Follower's timer expires without receiving a heartbeat, it becomes a Candidate. |
| **Majority Vote** | A Candidate must receive votes from a strict majority ( $>50\%$) of the cluster nodes to become the Leader. |

---

##  Leader Election Steps (5-Node Cluster Example)

The following sequence of steps demonstrates how a new leader is elected in a 5-node cluster (Nodes 1-5).

### Step 1: Initial State - All Followers

All nodes start in the **Follower** state, expecting periodic heartbeats from a Leader. They are all currently in **Term 0**.
*In the absence of a leader, all nodes are simply waiting for a heartbeat.*

**Image Reference:** `![Initial State](images/RAFT-images/RAFT-1.PNG)`
*(Depicts all 5 nodes as blue 'Followers' in Term: 0)*

### Step 2: Election Timeout - Node 3 Becomes Candidate

Node 3's randomized **Election Timeout** expires.

1.  Node 3 **increments its Term** from 0 to **1**.
2.  It transitions from a **Follower** to a **Candidate** (yellow).
3.  It votes for itself (Votes: 1).

**Image Reference:** `![Election Timeout - Node 3 Becomes Candidate](images/RAFT-images/RAFT-2.PNG)`
*(Depicts Node 3 as the yellow 'Candidate' in Term: 1, while others remain 'Followers' in Term: 0)*

### Step 3: Vote Requests Sent

The Candidate (Node 3) immediately sends **RequestVote RPCs** (Remote Procedure Calls) to all other nodes in the cluster. Upon receiving a request from a higher term (Term 1), the other nodes also transition to **Term 1**.

> *Node 3 asks the other nodes to acknowledge its bid for leadership in the new Term 1.*

**Image Reference:** `![Vote Requests Sent](images/RAFT-images/RAFT-3.PNG)`
*(Depicts Node 3 as 'Candidate' in Term: 1, and all other nodes as 'Followers' in Term: 1)*

### Step 4: Nodes Cast Votes

The other Followers (Nodes 1, 2, and 4) receive the RequestVote RPC. Since Node 3's Term is valid, and they have not yet voted in Term 1, they **grant their vote** to Node 3.

* Node 3 now has **4 votes** (from itself, 1, 2, and 4).
* Node 5 remains a Follower but has also updated its Term to 1.

**Image Reference:** `![Nodes Cast Votes](images/RAFT-images/RAFT-4.PNG)`
*(Depicts all Followers now recording a vote for Node 3)*

### Step 5: Node 3 Wins Election

Node 3 receives votes from the **majority** of the cluster (4 out of 5 nodes).

1.  Node 3 **transitions from a Candidate to the new Leader** (green).
2.  All nodes in the cluster are now stably in **Term 1**.

**Image Reference:** `![Node 3 Wins Election](images/RAFT-images/RAFT-5.PNG)`
*(Depicts Node 3 as the green 'Leader' with a crown, and all others as 'Followers', all in Term: 1)*

### Step 6: Leader Sends Heartbeats

The new Leader (Node 3) immediately begins sending **periodic Heartbeats** (AppendEntries RPCs with no log entries) to all other Followers.

> *These heartbeats maintain leadership by resetting the Followers' Election Timers and preventing new elections.*

**Image Reference:** `![Leader Sends Heartbeats](images/RAFT-images/RAFT-6.PNG)`
*(Depicts the final stable state: Node 3 is the 'Leader', and nodes 1, 2, 4, 5 are 'Followers', all in Term: 1)*

---

##  Key Takeaways

* **Consistency:** The use of **Term Numbers** ensures nodes only follow a Leader with an equal or higher Term.
* **Availability:** The **Randomized Election Timeout** minimizes the chance of a vote split, ensuring a timely leader is chosen.
* **Fault-Tolerance:** The requirement for a **Majority Vote** means the system can tolerate the failure of up to $N/2$ nodes without losing the ability to elect a leader and make progress.
