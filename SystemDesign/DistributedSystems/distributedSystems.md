# System Design: Distributed Systems

Comprehensive guide to distributed systems fundamentals, CAP theorem, consistency models, consensus algorithms, and real-world patterns with decision frameworks.

## Table of Contents
- [Distributed Systems Fundamentals](#distributed-systems-fundamentals)
- [CAP Theorem](#cap-theorem)
- [Consistency Models](#consistency-models)
- [Consensus Algorithms](#consensus-algorithms)
- [Replication Strategies](#replication-strategies)
- [Failure Handling](#failure-handling)
- [Real-World Architectures](#real-world-architectures)

---

## Distributed Systems Fundamentals

### What is a Distributed System?

**Definition:**
> A distributed system is a collection of independent computers that appear to users as a single coherent system.

**Why Distribute?**

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│          WHY BUILD DISTRIBUTED SYSTEMS? DECISION TREE              │
│                                                                     │
│  Do you need to handle > 10K concurrent users?                     │
│  ├── NO  → Single server might be sufficient                       │
│  └── YES → Continue                                                 │
│                                                                     │
│  Does your data exceed single machine capacity (>1TB)?             │
│  ├── YES → Must distribute (vertical scaling limit reached)        │
│  └── NO  → Continue                                                 │
│                                                                     │
│  Do users access system from multiple geographic regions?          │
│  ├── YES → Distribute for low latency (CDN, regional replicas)    │
│  └── NO  → Continue                                                 │
│                                                                     │
│  Can you tolerate single point of failure?                         │
│  ├── NO  → Must distribute for high availability                   │
│  └── YES → Single server acceptable                                │
│                                                                     │
│  Warning: Distributed systems add complexity!                       │
│  - Network failures                                                 │
│  - Partial failures                                                 │
│  - Clock synchronization issues                                     │
│  - Debugging difficulty                                             │
│  Only distribute when benefits outweigh costs!                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Challenges

**The Eight Fallacies of Distributed Computing:**
```
1. The network is reliable          ❌ Networks fail constantly
2. Latency is zero                  ❌ Speed of light is finite
3. Bandwidth is infinite            ❌ Limited bandwidth
4. The network is secure            ❌ Security breaches happen
5. Topology doesn't change          ❌ Nodes join/leave
6. There is one administrator       ❌ Multiple teams, orgs
7. Transport cost is zero           ❌ Serialization has cost
8. The network is homogeneous       ❌ Different hardware, OS
```

**Visual:**
```
Single Server (Simple):
┌─────────────┐
│ Application │
│ Database    │
│ Files       │
└─────────────┘
All on one machine

Failure: Server down = Everything down
Fix: Restart server
Debug: Check logs on one machine

Distributed System (Complex):
        ┌──────────┐
        │Load Bal  │
        └────┬─────┘
    ┌────────┼────────┐
    ↓        ↓        ↓
┌────────┐┌────────┐┌────────┐
│ App 1  ││ App 2  ││ App 3  │
└───┬────┘└───┬────┘└───┬────┘
    └─────┬────┴────┬────┘
          ↓         ↓
    ┌─────────┐ ┌─────────┐
    │  DB 1   │ │  DB 2   │
    └─────────┘ └─────────┘

Failure: One app node down? Others continue
         Network partition? Split brain!
         Clock skew? Timestamp conflicts!
Fix: Which node? What caused it? Cascade?
Debug: Logs scattered across machines
       Trace requests through system
```

---

## CAP Theorem

### The Core Principle

**CAP Theorem (Brewer's Theorem):**
> In presence of a network partition, a distributed system must choose between Consistency and Availability.

**The Three Properties:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                       CAP THEOREM EXPLAINED                         │
│                                                                     │
│  C = Consistency                                                     │
│  ├─ All nodes see the same data at the same time                   │
│  ├─ Read after write returns latest value                          │
│  └─ Strong consistency: Linearizability                            │
│                                                                     │
│  A = Availability                                                    │
│  ├─ Every request gets a response (no errors)                      │
│  ├─ System remains operational                                      │
│  └─ No guarantee response has latest data                          │
│                                                                     │
│  P = Partition Tolerance                                            │
│  ├─ System works despite network failures                          │
│  ├─ Nodes can't communicate                                        │
│  └─ MUST have (networks always fail!)                              │
│                                                                     │
│  Reality: You get 2 out of 3                                        │
│  Since P is mandatory → Choose C or A                               │
└─────────────────────────────────────────────────────────────────────┘
```

### CAP Decision Framework

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              CONSISTENCY VS AVAILABILITY DECISION                   │
│                                                                     │
│  What kind of system are you building?                             │
│                                                                     │
│  Financial/Banking System:                                          │
│  ├─ Wrong balance = Lost money                                     │
│  ├─ Choose: CONSISTENCY (CP)                                       │
│  ├─ Database: PostgreSQL (strong consistency)                      │
│  └─ Accept: System unavailable during partition                    │
│                                                                     │
│  E-commerce Product Catalog:                                        │
│  ├─ Stale price OK for 5 seconds                                   │
│  ├─ Choose: AVAILABILITY (AP)                                      │
│  ├─ Database: Cassandra (eventual consistency)                     │
│  └─ Accept: Users might see old prices briefly                     │
│                                                                     │
│  Social Media Feed:                                                 │
│  ├─ Post not showing immediately OK                                │
│  ├─ Choose: AVAILABILITY (AP)                                      │
│  ├─ Database: DynamoDB (eventual consistency)                      │
│  └─ Accept: Followers see posts at different times                 │
│                                                                     │
│  Inventory Management:                                              │
│  ├─ Overselling = Customer complaints                              │
│  ├─ Choose: CONSISTENCY (CP)                                       │
│  ├─ Database: PostgreSQL with row locking                          │
│  └─ Accept: "Out of stock" during network issues                   │
│                                                                     │
│  Key Question to Ask:                                               │
│  "What happens if user sees stale data?"                            │
│  ├─ Catastrophic (money loss, safety) → CONSISTENCY                │
│  ├─ Annoying but OK → AVAILABILITY                                 │
│  └─ Depends on feature → HYBRID (different per feature)            │
└─────────────────────────────────────────────────────────────────────┘
```

### Real-World CAP Example

**Scenario: Bank Transfer ($100 from Alice to Bob)**

**CP System (Consistent + Partition Tolerant):**
```
Normal Operation:
┌──────────┐         ┌──────────┐
│ Node A   │←──ok──→ │ Node B   │
│ Alice:   │         │ Bob:     │
│ $1000    │         │ $500     │
└──────────┘         └──────────┘

Request: Transfer $100 Alice → Bob

Step 1: Lock both accounts
Step 2: Deduct $100 from Alice
Step 3: Add $100 to Bob
Step 4: Commit on both nodes
Step 5: Unlock accounts

Result: Alice: $900, Bob: $600 ✓

Network Partition:
┌──────────┐    X    ┌──────────┐
│ Node A   │← fail →│ Node B   │
│ Alice:   │         │ Bob:     │
│ $1000    │         │ $500     │
└──────────┘         └──────────┘

Request: Transfer $100 Alice → Bob

Response: ❌ ERROR - "System temporarily unavailable"
Why? Can't guarantee both nodes see same data
Better: Reject than allow inconsistency

User Experience:
- Sees error message
- Must retry later
- Annoying but safe

Bank's Choice: CONSISTENCY
Reason: Wrong balance = Regulatory violation
```

**AP System (Available + Partition Tolerant):**
```
Normal Operation:
┌──────────┐         ┌──────────┐
│ Node A   │←──ok──→ │ Node B   │
│ Posts:   │         │ Posts:   │
│ [1,2,3]  │         │ [1,2,3]  │
└──────────┘         └──────────┘

Request: Add post #4

Both nodes accept writes independently
Result: Posts: [1,2,3,4] ✓

Network Partition:
┌──────────┐    X    ┌──────────┐
│ Node A   │← fail →│ Node B   │
│ Posts:   │         │ Posts:   │
│ [1,2,3]  │         │ [1,2,3]  │
└──────────┘         └──────────┘

User Alice → Node A: Add post #4
User Bob   → Node B: Add post #5

Node A: [1,2,3,4]
Node B: [1,2,3,5]

Different views! But both users got success response ✓

When partition heals:
Conflict resolution (last-write-wins):
Final: [1,2,3,4,5]

User Experience:
- Both users succeeded immediately
- No error messages
- Might see different feeds temporarily

Social Media's Choice: AVAILABILITY
Reason: Brief inconsistency OK, uptime critical
```

### PACELC Extension

**PACELC = More Complete Model:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│                    PACELC THEOREM DECISION                          │
│                                                                     │
│  IF Partition (P):                                                  │
│  ├─ Trade-off between Availability (A) and Consistency (C)         │
│  │  (This is CAP theorem)                                          │
│  │                                                                  │
│  ELSE (E) - Normal operation (no partition):                       │
│  └─ Trade-off between Latency (L) and Consistency (C)              │
│                                                                     │
│  Four Combinations:                                                 │
│                                                                     │
│  PA/EC (Prefer Availability, Accept Higher Latency):               │
│  ├─ Example: DynamoDB                                              │
│  ├─ Partition: Stay available                                      │
│  ├─ Normal: Strong consistency (slower)                            │
│  └─ Use: When availability critical, consistency needed            │
│                                                                     │
│  PA/EL (Prefer Availability and Low Latency):                      │
│  ├─ Example: Cassandra, Dynamo                                     │
│  ├─ Partition: Stay available                                      │
│  ├─ Normal: Eventual consistency (faster)                          │
│  └─ Use: High-traffic, user-facing apps                            │
│                                                                     │
│  PC/EC (Prefer Consistency, Accept Higher Latency):                │
│  ├─ Example: HBase, BigTable                                       │
│  ├─ Partition: Sacrifice availability                              │
│  ├─ Normal: Strong consistency (slower)                            │
│  └─ Use: Analytics, strong consistency required                    │
│                                                                     │
│  PC/EL (Prefer Consistency but Low Latency):                       │
│  ├─ Example: PostgreSQL with replication                           │
│  ├─ Partition: Sacrifice availability                              │
│  ├─ Normal: Compromise between consistency and speed               │
│  └─ Use: Balanced workloads                                        │
│                                                                     │
│  Decision Process:                                                  │
│  1. Can you tolerate inconsistency during partition?               │
│  2. Can you tolerate higher latency in normal operation?           │
│  3. Choose system based on answers                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Consistency Models

### Consistency Spectrum

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│                CONSISTENCY MODEL SPECTRUM                           │
│                                                                     │
│  STRONG ←──────────────────────────────────────────────→ WEAK      │
│                                                                     │
│  Linearizability (Strongest)                                        │
│  ├─ Operations appear instantaneous                                │
│  ├─ Total ordering of all operations                               │
│  ├─ Reads see most recent write                                    │
│  ├─ Cost: Highest latency, lower availability                      │
│  └─ Use: Financial transactions, inventory                         │
│                                                                     │
│  Sequential Consistency                                             │
│  ├─ All operations in some sequential order                        │
│  ├─ Same order across all nodes                                    │
│  ├─ Cost: High latency                                             │
│  └─ Use: Distributed databases needing ordering                    │
│                                                                     │
│  Causal Consistency                                                 │
│  ├─ Causally related writes seen in order                          │
│  ├─ Concurrent writes can be seen differently                      │
│  ├─ Cost: Medium latency                                           │
│  └─ Use: Social feeds, collaborative editing                       │
│                                                                     │
│  Eventual Consistency (Weakest)                                     │
│  ├─ All replicas converge eventually                               │
│  ├─ No ordering guarantees                                         │
│  ├─ Cost: Lowest latency, highest availability                     │
│  └─ Use: DNS, CDN, caching layers                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Consistency Model Selection

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│           CONSISTENCY MODEL SELECTION DECISION                      │
│                                                                     │
│  Does every read MUST return the most recent write?                │
│  ├── YES → Need Strong Consistency                                 │
│  │   ├─ Can tolerate higher latency?                              │
│  │   │   ├─ YES → Linearizability                                 │
│  │   │   └─ NO  → Consider if truly need strong consistency      │
│  │   └─ Examples: Banking, ticket booking, inventory              │
│  └── NO  → Can use Weaker Consistency                              │
│                                                                     │
│  Do causally related operations need ordering?                      │
│  ├── YES → Causal Consistency                                      │
│  │   └─ Examples: Social media (comment after post),              │
│  │       collaborative editing (Git commits)                       │
│  └── NO  → Continue                                                 │
│                                                                     │
│  Can users tolerate seeing stale data temporarily?                  │
│  ├── YES → Eventual Consistency                                    │
│  │   ├─ How stale is acceptable?                                  │
│  │   │   ├─ Seconds: Use TTL cache                                │
│  │   │   ├─ Minutes: DNS-style propagation                        │
│  │   │   └─ Hours: Batch reconciliation                          │
│  │   └─ Examples: Product catalog, user profiles, analytics       │
│  └── NO  → Need stronger consistency                               │
│                                                                     │
│  What's your read/write ratio?                                      │
│  ├── 95%+ reads → Eventual consistency with caching               │
│  ├── 50/50 → Causal consistency                                    │
│  └── Write-heavy → Need conflict resolution strategy              │
│                                                                     │
│  Real-World Pattern: Hybrid Approach                               │
│  ├─ Critical data (payments): Strong consistency                   │
│  ├─ User data (profiles): Eventual consistency                     │
│  └─ Social data (feeds): Causal consistency                        │
└─────────────────────────────────────────────────────────────────────┘
```

### Consistency Examples

**Linearizability (Strong):**
```
Timeline of operations:

Client A: WRITE(x, 1) starts
Client A: WRITE(x, 1) completes at T1
Client B: READ(x) at T2 (after T1) → MUST return 1
Client C: READ(x) at T3 (after T2) → MUST return 1

Visual:
T0────────T1────────T2────────T3────────→
   Write(1)   ✓     Read→1    Read→1

All reads after write see the new value
Real-time ordering preserved
```

**Causal Consistency:**
```
Social Media Example:

Alice: POST "Going to Paris!" (Event A)
Bob sees Event A
Bob: COMMENT "Have fun!" on Event A (Event B) → Causally dependent on A

Causal Guarantee:
- Anyone who sees Event B MUST also see Event A
- Can't see comment without seeing original post

Visual:
Alice: [Post] ────────────────→
         ↓
Bob:   [Comment] ──────────→
         ↓
Charlie sees:
✓ Post then Comment (correct)
❌ Comment then Post (impossible - causal violation)

Non-causal events (concurrent):
Alice: POST "Paris!"
Carol: POST "Nice weather!"
Different users might see in different order (OK!)
```

**Eventual Consistency:**
```
DNS Example:

T0: Update DNS: example.com → 1.2.3.4
T1: Server A sees new IP (1.2.3.4)
T2: Server B still has old IP (5.6.7.8)
T3: Server C gets new IP (1.2.3.4)
T5: Server B finally updates (1.2.3.4)

Eventually (T5): All servers have 1.2.3.4 ✓

During T1-T5: Different users see different IPs
Acceptable: DNS changes propagate slowly
```

---

## Consensus Algorithms

### Why Consensus Matters

**The Problem:**
```
Distributed systems need to agree on:
- Which node is the leader?
- What's the next value to write?
- Has a transaction committed?
- What's the current state?

Without consensus:
┌────────┐  ┌────────┐  ┌────────┐
│ Node A │  │ Node B │  │ Node C │
│ Leader │  │ Leader │  │ Leader │
└────────┘  └────────┘  └────────┘
     ↑           ↑           ↑
Split-brain! Multiple leaders!
Conflicting decisions!
Data corruption!

With consensus:
┌────────┐  ┌────────┐  ┌────────┐
│ LEADER │  │Follower│  │Follower│
└────────┘  └────────┘  └────────┘
     ↑           ↑           ↑
All agree on one leader ✓
Coordinated decisions ✓
```

### Consensus Algorithm Selection

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│          CONSENSUS ALGORITHM SELECTION DECISION                     │
│                                                                     │
│  What's your primary concern?                                       │
│                                                                     │
│  Ease of Understanding & Implementation:                            │
│  ├─ Algorithm: RAFT                                                │
│  ├─ Why: Designed explicitly for understandability                 │
│  ├─ Decomposed: Leader election + Log replication + Safety         │
│  ├─ Documentation: Excellent (visualizations, papers, talks)       │
│  └─ Use: Most new systems (etcd, Consul, CockroachDB)             │
│                                                                     │
│  Proven Track Record (Battle-Tested):                              │
│  ├─ Algorithm: PAXOS (Multi-Paxos)                                │
│  ├─ Why: Used in production for 20+ years                         │
│  ├─ Systems: Google Chubby, Spanner, Azure Storage                │
│  ├─ Drawback: Notoriously difficult to understand                 │
│  └─ Use: When need proven reliability, have expert team           │
│                                                                     │
│  Ordering Guarantees (Total Order):                                │
│  ├─ Algorithm: ZAB (ZooKeeper Atomic Broadcast)                   │
│  ├─ Why: Guarantees message ordering across leader changes        │
│  ├─ Systems: Apache ZooKeeper, Apache Kafka (older versions)      │
│  └─ Use: Coordination services, distributed locks                 │
│                                                                     │
│  Performance (Write-Heavy):                                         │
│  ├─ Algorithm: EPaxos (Egalitarian Paxos)                         │
│  ├─ Why: No single leader bottleneck                              │
│  ├─ Trade-off: More complex, less proven                          │
│  └─ Use: Research, high-throughput requirements                   │
│                                                                     │
│  Byzantine Fault Tolerance (Untrusted Nodes):                      │
│  ├─ Algorithm: PBFT or Tendermint                                 │
│  ├─ Why: Tolerate malicious/lying nodes                           │
│  ├─ Cost: 3F+1 nodes to tolerate F failures (vs 2F+1)            │
│  └─ Use: Blockchain, permissioned networks                        │
│                                                                     │
│  Default Recommendation:                                            │
│  └─ Use RAFT unless you have specific reason not to               │
│      ✓ Well-documented                                            │
│      ✓ Easy to debug                                              │
│      ✓ Many implementations                                        │
│      ✓ Good performance                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### Raft Consensus Deep Dive

**Raft Components:**
```
┌─────────────────────────────────────────────────────────────────────┐
│                      RAFT ARCHITECTURE                              │
│                                                                     │
│  Node States:                                                       │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                    │
│  │ FOLLOWER │ ←→ │CANDIDATE │ ←→ │  LEADER  │                    │
│  └──────────┘    └──────────┘    └──────────┘                    │
│      ↑               ↑                ↓                            │
│      │           Election         Commands                         │
│      │           Timeout           from clients                    │
│      └───────────────────────────────┘                            │
│                                                                     │
│  1. Leader Election:                                                │
│     ├─ Followers have timeout (150-300ms random)                   │
│     ├─ Timeout → Become candidate                                  │
│     ├─ Request votes from other nodes                              │
│     ├─ Majority votes → Become leader                              │
│     └─ Only one leader per term                                    │
│                                                                     │
│  2. Log Replication:                                                │
│     ├─ Leader receives client request                              │
│     ├─ Appends to its log                                          │
│     ├─ Sends AppendEntries RPC to followers                        │
│     ├─ Waits for majority acknowledgment                           │
│     └─ Commits entry, applies to state machine                     │
│                                                                     │
│  3. Safety:                                                         │
│     ├─ Leader must have all committed entries                      │
│     ├─ Only up-to-date candidates can win election                 │
│     ├─ Leader never overwrites committed entries                   │
│     └─ If entry committed, all future leaders have it              │
└─────────────────────────────────────────────────────────────────────┘
```

**Raft Timeline Example:**
```
5-node cluster: A, B, C, D, E

T0: Normal Operation
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│LEADER A│  │Follow B│  │Follow C│  │Follow D│  │Follow E│
└────────┘  └────────┘  └────────┘  └────────┘  └────────┘

Client → A: Write "x=1"
A appends to log: [x=1]
A → B,C,D,E: AppendEntries [x=1]
B,C,D,E: OK ✓
A: Majority received (5/5), commit!
A applies x=1 to state machine
A → Client: Success ✓

T1: Leader Fails
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│ DEAD A │  │Follow B│  │Follow C│  │Follow D│  │Follow E│
└────────┘  └────────┘  └────────┘  └────────┘  └────────┘

B's timeout expires (no heartbeat from A)
B → CANDIDATE
B: Election term = 2
B → C,D,E: RequestVote

C: YES (vote for B)
D: YES (vote for B)
E: YES (vote for B)

B got majority (3/4 live nodes)
B → LEADER ✓

T2: New Leader Accepts Writes
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│ DEAD A │  │LEADER B│  │Follow C│  │Follow D│  │Follow E│
└────────┘  └────────┘  └────────┘  └────────┘  └────────┘

Client → B: Write "y=2"
B continues normal operation
System remains available! ✓
```

**Raft Safety Guarantee:**
```
Key Property: Election Safety

Scenario: After A committed [x=1], crashes before telling E

┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│  A     │  │  B     │  │  C     │  │  D     │  │  E     │
│ [x=1]✓ │  │ [x=1]✓ │  │ [x=1]✓ │  │ [x=1]✓ │  │ [ ]    │
└────────┘  └────────┘  └────────┘  └────────┘  └────────┘
  DEAD        ↑                                      ↑
              Has x=1                          Missing x=1

Election between B and E:

Can E become leader?
❌ NO! E's log is NOT up-to-date
✓ B has longer log, wins election
✓ B replicates [x=1] to E
✓ Committed entry never lost!

Raft Rule: Only candidates with most up-to-date log can win
```

### Consensus Trade-offs

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              CONSENSUS ALGORITHM TRADE-OFFS                         │
│                                                                     │
│  Raft:                                                              │
│  ✅ Easy to understand and implement                                │
│  ✅ Strong leader simplifies log replication                        │
│  ✅ Clear separation of concerns                                    │
│  ❌ Leader is bottleneck (all writes go through leader)            │
│  ❌ Leader election adds latency during failures                    │
│  ❌ Write latency: 1-2 RTTs (round-trip times)                     │
│                                                                     │
│  Paxos (Multi-Paxos):                                              │
│  ✅ Proven correct for 20+ years                                    │
│  ✅ Battle-tested in production (Google, Microsoft)                │
│  ✅ Flexible (can work without strong leader)                       │
│  ❌ Extremely difficult to understand                               │
│  ❌ Hard to implement correctly                                     │
│  ❌ Many subtle edge cases                                          │
│                                                                     │
│  Performance Comparison (5 nodes):                                  │
│                                                                     │
│  Write Latency:                                                     │
│  ├─ Single datacenter: 1-5ms (both similar)                        │
│  ├─ Cross-datacenter: 50-200ms (both similar)                      │
│  └─ Rule: Network latency dominates, not algorithm                 │
│                                                                     │
│  Throughput:                                                        │
│  ├─ Raft: 10K-100K ops/sec                                         │
│  ├─ Paxos: 10K-100K ops/sec                                        │
│  └─ Bottleneck: Leader network bandwidth                           │
│                                                                     │
│  Fault Tolerance (5 nodes):                                         │
│  ├─ Can tolerate: 2 failures                                        │
│  ├─ Need majority: 3/5 nodes                                        │
│  ├─ Formula: Tolerate F failures = Need 2F+1 nodes                 │
│  └─ Same for both algorithms                                        │
│                                                                     │
│  Recommendation:                                                    │
│  ├─ New project → Use Raft                                         │
│  ├─ Must be battle-tested → Use Paxos                              │
│  └─ Need Byzantine tolerance → Use PBFT/Tendermint                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Replication Strategies

### Replication Decision Framework

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              REPLICATION STRATEGY SELECTION                         │
│                                                                     │
│  What's your primary goal?                                          │
│                                                                     │
│  High Availability (System stays up during failures):              │
│  ├─ Strategy: Multi-Master Replication                            │
│  ├─ Pattern: Any node accepts writes                              │
│  ├─ Benefit: No single point of failure                           │
│  ├─ Cost: Conflict resolution needed                              │
│  └─ Example: Cassandra, DynamoDB                                  │
│                                                                     │
│  Strong Consistency (All nodes see same data):                     │
│  ├─ Strategy: Single-Master Replication                           │
│  ├─ Pattern: One primary, multiple replicas                       │
│  ├─ Benefit: No write conflicts                                   │
│  ├─ Cost: Primary is bottleneck and single point of failure       │
│  └─ Example: PostgreSQL with streaming replication                │
│                                                                     │
│  Low Latency (Fast reads worldwide):                               │
│  ├─ Strategy: Geographic Replication                              │
│  ├─ Pattern: Replicas in multiple regions                         │
│  ├─ Benefit: Read from nearby replica                             │
│  ├─ Cost: Cross-region replication lag                            │
│  └─ Example: Global CDN, DynamoDB Global Tables                   │
│                                                                     │
│  Read Scalability (Handle many reads):                             │
│  ├─ Strategy: Read Replicas                                       │
│  ├─ Pattern: One primary (writes), many replicas (reads)          │
│  ├─ Benefit: Horizontal read scaling                              │
│  ├─ Cost: Replication lag acceptable                              │
│  └─ Example: MySQL with 5+ read replicas                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Synchronous vs Asynchronous Replication

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│       SYNCHRONOUS VS ASYNCHRONOUS REPLICATION DECISION              │
│                                                                     │
│  Synchronous Replication:                                           │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ Write Flow:                                                   │ │
│  │ 1. Client → Primary: Write x=1                               │ │
│  │ 2. Primary → Replica: Replicate x=1                          │ │
│  │ 3. Replica → Primary: ACK received                           │ │
│  │ 4. Primary → Client: Success ✓                               │ │
│  │                                                               │ │
│  │ ✅ Pros:                                                       │ │
│  │   - No data loss (replica has data before acknowledgment)    │ │
│  │   - Strong consistency guarantee                             │ │
│  │   - Read from replica is always up-to-date                   │ │
│  │                                                               │ │
│  │ ❌ Cons:                                                       │ │
│  │   - Higher write latency (wait for replica)                  │ │
│  │   - Availability: Primary blocks if replica down             │ │
│  │   - Cross-region: 100-200ms penalty                          │ │
│  │                                                               │ │
│  │ Use When:                                                     │ │
│  │   - Data loss unacceptable (financial transactions)          │ │
│  │   - Must read your own writes (strong consistency)           │ │
│  │   - Regulatory requirements (compliance)                     │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  Asynchronous Replication:                                          │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ Write Flow:                                                   │ │
│  │ 1. Client → Primary: Write x=1                               │ │
│  │ 2. Primary → Client: Success ✓ (immediately!)                │ │
│  │ 3. Primary → Replica: Replicate x=1 (in background)          │ │
│  │                                                               │ │
│  │ ✅ Pros:                                                       │ │
│  │   - Low write latency (no waiting)                           │ │
│  │   - High availability (primary never blocks)                 │ │
│  │   - Works across continents                                  │ │
│  │                                                               │ │
│  │ ❌ Cons:                                                       │ │
│  │   - Data loss possible (if primary fails before replication) │ │
│  │   - Replication lag (replicas behind)                        │ │
│  │   - Stale reads possible                                     │ │
│  │                                                               │ │
│  │ Use When:                                                     │ │
│  │   - Low latency critical (user experience)                   │ │
│  │   - Brief inconsistency acceptable (social feeds)            │ │
│  │   - Global distribution needed                               │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  Semi-Synchronous (Hybrid):                                         │
│  ├─ Wait for 1 replica (not all) before ACK                       │
│  ├─ Balance: Some durability, better latency                       │
│  └─ Example: PostgreSQL with synchronous_standby_names=1          │
└─────────────────────────────────────────────────────────────────────┘
```

### Conflict Resolution Strategies

**Multi-Master Conflicts:**
```
Problem: Same data written to different nodes

Example - Shopping Cart:
┌────────────┐              ┌────────────┐
│  Node A    │              │  Node B    │
│  Cart: []  │              │  Cart: []  │
└────────────┘              └────────────┘

User Alice → Node A: Add Item X
User Alice → Node B: Add Item Y

After sync:
Node A: Cart: [X]           Node B: Cart: [Y]

Conflict! Which is correct?

┌─────────────────────────────────────────────────────────────────────┐
│           CONFLICT RESOLUTION STRATEGY DECISION                     │
│                                                                     │
│  Last-Write-Wins (LWW):                                             │
│  ├─ Use: Timestamp to determine winner                            │
│  ├─ Pro: Simple, deterministic                                    │
│  ├─ Con: Data loss (one write discarded)                          │
│  ├─ Example: Cassandra default                                    │
│  └─ Use when: Overwrites acceptable (user preferences)            │
│                                                                     │
│  Vector Clocks:                                                     │
│  ├─ Track: Causality between writes                               │
│  ├─ Pro: Detects concurrent writes                                │
│  ├─ Con: Complex, requires application logic                       │
│  ├─ Example: Dynamo, Riak                                         │
│  └─ Use when: Need to preserve concurrent updates                 │
│                                                                     │
│  CRDTs (Conflict-Free Replicated Data Types):                      │
│  ├─ Guarantee: Merges always converge to same state              │
│  ├─ Pro: No conflicts, automatic resolution                        │
│  ├─ Con: Limited data types, memory overhead                      │
│  ├─ Example: Collaborative editing (Google Docs)                  │
│  └─ Use when: Concurrent edits are common                         │
│                                                                     │
│  Application-Level Resolution:                                      │
│  ├─ Let: Application decide how to merge                          │
│  ├─ Pro: Domain-specific logic                                    │
│  ├─ Con: Complexity in application code                           │
│  ├─ Example: Shopping cart: Merge both items                      │
│  └─ Use when: Business logic must decide                          │
│                                                                     │
│  Shopping Cart Example Resolution:                                 │
│  ├─ LWW: Cart = [Y] (latest wins, Item X lost!)                  │
│  ├─ Merge: Cart = [X, Y] (union of both) ✓ Best!                │
│  └─ App decides: Merge both, show to user for confirmation        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Failure Handling

### Failure Types

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│                  DISTRIBUTED SYSTEM FAILURES                        │
│                                                                     │
│  Crash Failure (Node dies):                                         │
│  ├─ Symptom: Node stops responding                                │
│  ├─ Detection: Heartbeat timeout (no response)                    │
│  ├─ Impact: Other nodes must compensate                           │
│  └─ Recovery: Restart node, resync data                           │
│                                                                     │
│  Network Partition (Split brain):                                   │
│  ├─ Symptom: Two groups of nodes can't communicate                │
│  ├─ Detection: Each group thinks other is down                    │
│  ├─ Impact: Both groups might elect leaders (dangerous!)          │
│  └─ Recovery: Use quorum to prevent split brain                   │
│                                                                     │
│  Byzantine Failure (Malicious/buggy node):                          │
│  ├─ Symptom: Node sends incorrect/conflicting data                │
│  ├─ Detection: Requires Byzantine fault tolerance                 │
│  ├─ Impact: Can corrupt data or mislead other nodes               │
│  └─ Recovery: Use PBFT, require 3F+1 nodes                        │
│                                                                     │
│  Slow Node (Performance degradation):                               │
│  ├─ Symptom: Node responds slowly but doesn't crash               │
│  ├─ Detection: Monitor latency, set timeouts                      │
│  ├─ Impact: Delays entire system if not handled                   │
│  └─ Recovery: Route around slow node, investigate                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Failure Detection Decision

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              FAILURE DETECTION STRATEGY                             │
│                                                                     │
│  How quickly must you detect failures?                              │
│  ├─ Immediate (< 1 second)                                         │
│  │   ├─ Method: Aggressive heartbeats (100ms intervals)           │
│  │   ├─ Cost: High network overhead                               │
│  │   └─ Use: Financial systems, real-time applications            │
│  │                                                                  │
│  ├─ Fast (1-10 seconds)                                            │
│  │   ├─ Method: Standard heartbeats (1-5s intervals)              │
│  │   ├─ Cost: Moderate overhead                                    │
│  │   └─ Use: Most distributed databases, web services             │
│  │                                                                  │
│  └─ Relaxed (10+ seconds)                                          │
│      ├─ Method: Periodic health checks                             │
│      ├─ Cost: Low overhead                                         │
│      └─ Use: Batch processing, analytics                           │
│                                                                     │
│  How do you handle false positives?                                 │
│  ├─ Single timeout → Declare failed                                │
│  │   └─ Risk: Network hiccup causes unnecessary failover          │
│  ├─ Multiple missed heartbeats (3-5)                               │
│  │   └─ Safer: Reduces false positives                            │
│  └─ Phi Accrual Failure Detector                                   │
│      └─ Sophisticated: Adapts to network conditions                │
│                                                                     │
│  Heartbeat Pattern:                                                 │
│  Leader                         Follower                            │
│     │────heartbeat (1s)────────→ │                                │
│     │←───ack───────────────────── │                                │
│     │────heartbeat (1s)────────→ │                                │
│     │←───ack───────────────────── │                                │
│     │                             │                                │
│     │  (network issue)            │                                │
│     │                             │                                │
│     │  (no ack after 3s)          │                                │
│     │  Declare follower DOWN!     │                                │
└─────────────────────────────────────────────────────────────────────┘
```

### Handling Split Brain

**The Problem:**
```
5-node cluster with network partition:

Normal:
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│LEADER A│─ │Follow B│─ │Follow C│─ │Follow D│─ │Follow E│
└────────┘  └────────┘  └────────┘  └────────┘  └────────┘

Network Partition:
┌────────┐  ┌────────┐  ┌────────┐  │  ┌────────┐  ┌────────┐
│LEADER A│─ │Follow B│─ │Follow C│  X  │Follow D│─ │Follow E│
└────────┘  └────────┘  └────────┘  │  └────────┘  └────────┘
     Group 1 (3 nodes)          Network    Group 2 (2 nodes)
                                 Partition

Without Quorum:
Group 1 thinks: D and E are down, A still leader
Group 2 thinks: A, B, C are down, elect new leader!

Result: TWO LEADERS! (split brain)
Both accept writes → Data divergence → DISASTER!

With Quorum (Majority Required):
Group 1: Has 3/5 nodes (majority ✓) → A remains leader
Group 2: Has 2/5 nodes (minority ✗) → Cannot elect leader

Group 1: Continues accepting writes ✓
Group 2: Rejects all writes (no quorum) ✓

Only one active leader! Data safe! ✓
```

**Quorum Decision:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│                    QUORUM SIZE DECISION                             │
│                                                                     │
│  How many nodes do you need?                                        │
│                                                                     │
│  Formula: N nodes can tolerate F failures                           │
│  ├─ Quorum size: (N/2) + 1                                         │
│  ├─ Max failures: (N-1)/2                                          │
│  └─ Must have: N ≥ 2F + 1                                          │
│                                                                     │
│  Examples:                                                          │
│  ├─ 3 nodes: Tolerate 1 failure (need 2/3 for quorum)             │
│  ├─ 5 nodes: Tolerate 2 failures (need 3/5 for quorum)            │
│  ├─ 7 nodes: Tolerate 3 failures (need 4/7 for quorum)            │
│  └─ 9 nodes: Tolerate 4 failures (need 5/9 for quorum)            │
│                                                                     │
│  Why odd numbers?                                                   │
│  ├─ 4 nodes: Tolerate 1 failure (need 3/4)                        │
│  ├─ 5 nodes: Tolerate 2 failures (need 3/5)                        │
│  └─ Same quorum size (3), but 5 nodes tolerate more failures!     │
│                                                                     │
│  Recommendation:                                                    │
│  ├─ Development/Testing: 3 nodes                                   │
│  ├─ Production (standard): 5 nodes                                 │
│  ├─ Critical systems: 7 nodes                                      │
│  └─ Rarely need: > 7 nodes (diminishing returns)                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Architectures

### Google Spanner

**Design Decisions:**
```markdown
Requirements:
- Global scale (billions of users)
- Strong consistency (financial data)
- High availability (99.999%)
- ACID transactions

Key Innovations:
1. TrueTime API
   ├─ GPS + atomic clocks
   ├─ Provides global time with bounded uncertainty
   └─ Enables external consistency across datacenters

2. Paxos for Replication
   ├─ Every shard uses Paxos group
   ├─ Synchronous replication across zones
   └─ Strong consistency within shard

3. Two-Phase Commit for Transactions
   ├─ Coordinator uses Paxos
   ├─ Handles cross-shard transactions
   └─ ACID guarantees globally

Trade-offs:
✅ Strong consistency globally
✅ ACID transactions
✅ High availability
❌ Higher write latency (cross-datacenter sync)
❌ Expensive (needs specialized hardware)
```

### Cassandra

**Design Decisions:**
```markdown
Requirements:
- Handle millions of writes/sec
- Global distribution
- Always available (AP system)
- Linear scalability

Key Design:
1. Dynamo-style Architecture
   ├─ Ring-based consistent hashing
   ├─ No single point of failure
   └─ Peer-to-peer (no leader)

2. Tunable Consistency
   ├─ Write: ONE, QUORUM, or ALL
   ├─ Read: ONE, QUORUM, or ALL
   └─ Application chooses per query

3. LSM Tree Storage
   ├─ Optimized for writes
   ├─ Memtable + SSTable
   └─ Background compaction

Trade-offs:
✅ Extremely high write throughput
✅ Linear scalability (add nodes = add capacity)
✅ Always available
❌ Eventual consistency (by default)
❌ Limited query flexibility (no joins)
❌ Conflict resolution required
```

### Comparison Matrix

```markdown
┌────────────────┬──────────┬───────────┬───────────┬───────────┐
│ System         │ Spanner  │ Cassandra │ PostgreSQL│ DynamoDB  │
├────────────────┼──────────┼───────────┼───────────┼───────────┤
│ CAP Choice     │   CP     │    AP     │    CP     │    AP     │
│ Consensus      │  Paxos   │   None    │   None    │   Paxos   │
│ Consistency    │ Strong   │  Eventual │  Strong   │ Eventual  │
│ Transactions   │ Global   │   None    │  Local    │  Limited  │
│ Write Latency  │  High    │   Low     │  Medium   │   Low     │
│ Scalability    │ Excellent│ Excellent │  Limited  │ Excellent │
│ Ops Complexity │   High   │  Medium   │   Low     │   Low     │
│ Best For       │ Financial│  IoT/Logs │  OLTP     │ Serverless│
└────────────────┴──────────┴───────────┴───────────┴───────────┘
```

---

## Quick Reference

### Decision Framework Summary

```markdown
Choose CP (Consistency + Partition Tolerance) when:
✅ Financial transactions
✅ Inventory management
✅ Booking systems
✅ Any system where wrong data = money/safety loss

Choose AP (Availability + Partition Tolerance) when:
✅ Social media feeds
✅ User profiles
✅ Product catalogs
✅ Analytics/logs
✅ IoT data

Consensus Algorithm:
✅ Default: Raft (easiest)
✅ Battle-tested: Paxos
✅ Ordering: ZAB (ZooKeeper)
✅ Byzantine: PBFT

Replication:
✅ Strong consistency: Synchronous replication
✅ High availability: Asynchronous replication
✅ Read scaling: Read replicas
✅ Global: Multi-region with conflict resolution
```

### Common Patterns

```markdown
Pattern: Read-Heavy Social Media
├─ Write: Asynchronous multi-master
├─ Read: Local replicas with CDN
├─ Consistency: Eventual
└─ Example: Twitter, Instagram

Pattern: E-Commerce Transactions
├─ Write: Synchronous to replicas
├─ Read: Read replicas with caching
├─ Consistency: Strong for orders, eventual for catalog
└─ Example: Amazon, Shopify

Pattern: Real-Time Analytics
├─ Write: Stream processing with Kafka
├─ Read: Pre-aggregated in OLAP DB
├─ Consistency: Eventual
└─ Example: Data dashboards, metrics

Pattern: Distributed Locking
├─ Consensus: Raft or Paxos
├─ Service: ZooKeeper, etcd, Consul
├─ Consistency: Linearizable
└─ Example: Leader election, configuration management
```

---

This comprehensive guide covers distributed systems fundamentals with decision trees for CAP theorem, consistency models, consensus algorithms, replication strategies, and real-world architectures.