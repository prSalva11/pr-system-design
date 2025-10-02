# 🚀 KRaft Consensus Mechanism: Controller Election and Metadata Management

The **KRaft (Kafka Raft)** consensus protocol is a critical evolution in Apache Kafka, replacing the reliance on ZooKeeper for metadata management and leader election. KRaft integrates the Raft algorithm directly into Kafka's Controller layer, simplifying the architecture and improving scalability and performance.

This guide illustrates the two main functions of KRaft: **Active Controller Election** and **Metadata Log Replication** in a cluster with 3 Controller Nodes and 2 Regular Broker Nodes.

---

## 🎯 Core KRaft Architecture

KRaft segments the cluster nodes into distinct roles:

| Component | Role | Description |
| :--- | :--- | :--- |
| **Controller Quorum (Voter Nodes)** | Elect a leader and replicate the metadata log. They follow the Raft protocol. | Nodes 1, 2, and 3 in this example. |
| **Active Controller (Leader)** | The single node elected by the quorum. It manages all cluster metadata (topics, partitions, etc.) and is the sole node that processes write requests. | A green node in the Quorum. |
| **Regular Brokers (Observer Nodes)** | They do *not* vote in elections. They fetch metadata updates from the Active Controller and apply them locally. | Nodes 4 and 5 in this example. |

---

## 🗳️ Active Controller Election Steps

The election process occurs only among the Controller Quorum (Voter Nodes).

### Step 1: Initial State - KRaft Cluster

All nodes start in a **Standby** state. Controller-eligible nodes (1-3) are part of the voting quorum. Regular Brokers (4-5) are observers only.

**Image Reference:** ![Initial State - KRaft Cluster](https://github.com/prSalva11/pr-system-design/raw/main/leader-consensus-algorithm-explanation/images/KRAFT-images/KRAFT-1.PNG)

### Step 2: Controller Election Begins

One of the Standby Controllers (Node 2) experiences an election timeout.

1.  Node 2 **increments its Term** to 1.
2.  It transitions to the **Candidate** state (yellow).
3.  Only **Controller-eligible nodes** can become candidates.

**Image Reference:** ![Controller Election Begins](https://github.com/prSalva11/pr-system-design/raw/main/leader-consensus-algorithm-explanation/images/KRAFT-images/KRAFT-2.PNG)

### Step 3: Voting Among Controllers

The Candidate (Node 2) sends RequestVote RPCs to the other quorum members (Nodes 1 and 3).

* Only Controller nodes (1, 2, 3) participate in the voting process.
* Node 2 wins the election by securing a majority vote (2/3 in this 3-node quorum).

**Image Reference:** ![Voting Among Controllers](https://github.com/prSalva11/pr-system-design/raw/main/leader-consensus-algorithm-explanation/images/KRAFT-images/KRAFT-3.PNG)

### Step 4: Active Controller Elected

Node 2 transitions to the **Active Controller** (green). It immediately starts managing metadata and coordinating the cluster via heartbeat AppendEntries RPCs.

**Image Reference:** ![Active Controller Elected](https://github.com/prSalva11/pr-system-design/raw/main/leader-consensus-algorithm-explanation/images/KRAFT-images/KRAFT-4.PNG)

---

## 📝 Metadata Management and Replication Steps

Once the Active Controller is established, it becomes the sole authority for cluster state changes (e.g., creating a topic, reassigning partitions).

### Step 5: Metadata Write Request

A Regular Broker (Node 4) or a client requests a change to the cluster state (e.g., creating a new topic). This request is always routed to the **Active Controller** (Node 2).

**Image Reference:** ![Metadata Write Request](https://github.com/prSalva11/pr-system-design/raw/main/leader-consensus-algorithm-explanation/images/KRAFT-images/KRAFT-5.PNG)

### Step 6: Log Replication to Quorum

The Active Controller converts the state change request into an entry in its **metadata log**. It then replicates this log entry to the **follower controllers** (Nodes 1 and 3). The Active Controller waits for an acknowledgment from a **quorum majority** (2/3) before proceeding.

> *This step ensures the metadata change is persistent and durable.*

**Image Reference:** ![Log Replication to Quorum](https://github.com/prSalva11/pr-system-design/raw/main/leader-consensus-algorithm-explanation/images/KRAFT-images/KRAFT-6.PNG)

### Step 7: Metadata Committed & Distributed

Once the log entry is replicated to the quorum, the metadata change is officially **committed**.

1.  The Active Controller applies the change locally.
2.  All nodes (Controller Followers and Regular Brokers) fetch the latest metadata updates from the metadata log.
3.  The Regular Brokers (Observers) are now **Metadata Synced** and can use the new information.

**Image Reference:** ![Metadata Committed & Distributed](https://github.com/prSalva11/pr-system-design/raw/main/leader-consensus-algorithm-explanation/images/KRAFT-images/KRAFT-7.PNG)

---

## 🔑 KRaft Benefits Over ZooKeeper

* **Simplified Architecture:** Removes the external ZooKeeper dependency, making deployment and operation simpler.
* **Faster Failover:** Leader election and metadata consistency are handled directly by the high-performance Raft implementation within Kafka, leading to much quicker Controller failover times.
* **Scalability:** Allows Kafka to handle larger clusters and more partitions efficiently.