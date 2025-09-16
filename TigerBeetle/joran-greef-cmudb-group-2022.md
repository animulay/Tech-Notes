YouTube video: [TigerBeetle: Magical Memory Tour! (Joran Dirk Greef)](https://www.youtube.com/watch?v=FyGukn77gqA) CMU Database Group, Nov 23, 2022

- Distributed database
- Designed to count business events OR balance moving between 2 or more parties at scale<br>
  Sample use-cases: financial transactions, in-app purchases, API billing, rate limiting, page views, page likes ...
- Open source (Apache-2.0 license)
- Double-Entry Accounting Primitives out-of-the box
- Unique Implementation characteristics e.g. Static Memory Allocation [^1]<br>
  Memory allocated at the time of initialization. No dynamic memory allocation during runtime.

#### Why Design a Ledger Database?

Business / Domain Requirements:
- Double-Entry Accounting at scale
- Two-Phase Commits Across Ledgers
- Tracking Money Movements within the System
- Tracking Money Movements between the Systems
- Strict Serializability and High Availability
- High Throughput and Low Latency

Design Goals:
1. 10X Safety - Adapted NASA's Ten Rules for Safety-Critical Code [^2]
3. 1000X Performance with 10X Less Hardware
4. Predictability
5. Transaction Velocity [^3]

A Typical Financial Transaction has

```
{
   id,
   date,
   description,
   debit_account_id,
   credit_account_id,
   amount,
   [user_data]
}
```

|Industry Trend Before TigerBeetle | TigerBeetle Goal |
|:---------------------------------|:-----------------|
| 10-20 DB Queries per Financial Transaction | 1 DB Query per 10K Financial Transactions |
| Client: 1 Financial Transaction per query | Client: 10K Financial Transactions in 1 Query (Batching) |

#### Important Iplementation Details:

- Replicated State Machine with Viewstamped Replication (VSR) Consensus Algorithm [^4] [^5]
  - Faster Fault Detection
  - Faster Fault Isolation
  - Can Predict the Next Leader
    - prioritize synchronous replication to the next predictive leader and asynchronous replication to others
- Ringo-Star Topology
  - Machines are arranged in a ring where the leader replicates to the next machine
  - Each machine sends an Ack to the leader
- Decouple Performance From Allocations
- Reduce Memcpy

To continue https://youtu.be/FyGukn77gqA?t=1845

#### References:
[^1]: [Hacker News](https://news.ycombinator.com/item?id=33192288) discussion thread about [A Database Without Dynamic Memory Allocation](https://tigerbeetle.com/blog/2022-10-12-a-database-without-dynamic-memory/), Phil Eaton, Oct 12, 2022
[^2]: [The Power of 10: Rules for Developing Safety-Critical Code](https://en.wikipedia.org/wiki/The_Power_of_10:_Rules_for_Developing_Safety-Critical_Code)
[^3]: [Evolution of Financial Exchange Architectures](https://www.infoq.com/presentations/financial-exchange-architecture/), Martin Thompson
[^4]: [SHOWTIME #28: Viewstamped Replication Made Famous](https://www.youtube.com/watch?v=_Jlikdtm4OA)<br>[tigerbeetle / viewstamped-replication-made-famous](https://github.com/tigerbeetle/viewstamped-replication-made-famous)
[^5]: [VSR Replication Code](https://github.com/tigerbeetle/tigerbeetle/blob/main/src/vsr/replica.zig)
