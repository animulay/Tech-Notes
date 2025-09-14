
## [TigerBeetle](https://tigerbeetle.com/): The Financial Transactions Database

[GitHub Repo](https://github.com/tigerbeetle/tigerbeetle)

--- 

YouTube video: [1000x: The Power of an Interface for Performance by Joran Dirk Greef](https://www.youtube.com/watch?v=yKgfk8lTQuE), Sep 1, 2025<br>
[Joran Greef](https://x.com/jorandirkgreef), Founder and CEO @TigerBeetle

---
---
YouTube video: [A New Era for Database Design with TigerBeetle](https://www.youtube.com/watch?v=ehYcCTHRyFs), InfoQ, Sept 21, 2023<br>
Joran Greef, Founder and CEO @TigerBeetle

- New open-source distributed database
- Designed to track the "Movement of Value" e.g. financial transactions - payments, trades
- Mission-Critical Safety and Performance
- Schema support for Balance Tracking via Double-Entry Accounting Primitives out-of-the-box
- Designed for High Availability (HA) with automated failover support

**What is Durability?**<br>
Once a database transaction has been acknowledge as committed to the user,<br>
it will remain committed, even in the event of a crash.

[https://danluu.com/file-consistency](https://danluu.com/file-consistency)

[All File Systems Are Not Created Equal:On the Complexity of Crafting Crash-Consistent Applications](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-pillai.pdf)

At least 3 ways that a database can be designed to write to disk:
1. [mmap()](https://linux.die.net/man/2/mmap)
   - [Are You Sure You Want to Use MMAP in Your Database Management System?](https://db.cs.cmu.edu/mmap-cidr2022/) 

2. [Direct I/O](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/5/html/global_file_system/s1-manage-direct-io) (O_DIRECT)
   - database takes the responsibility for working with the disk
   - bypasses the kernel page cache
  
3. Buffered I/O and [fsync()](https://linux.die.net/man/2/fsync)
   - outsource durability to kernel with buffered I/O
   - this strategy employed by Postgres and several other databases
  
   If fsync() returns an error (EIO), then the database has 3 options:<br>
   i. Ignore fsync() error<br>
   ii. Retry fsync() until success<br>
   iii. Crash and restart<br>

   The cracks in Buffered I/O strategy<br>
   a. Writing to cache instead of disk => You Lose Congetion Control<br>
   b. You Lose the Ability To Prioritize I/O<br>
   c. All or nothing => You Lose Fine-Grained Error Handling<br>
   d. You Lose Memory Bandwidth<br>

   • [PostgreSQL's fsync() surprise](https://lwn.net/Articles/752063/) By Jonathan Corbet, April 18, 2018

   • [PostgreSQL vs. fsync
How is it possible that PostgreSQL used fsync incorrectly for 20 years, and what we'll do about it.](https://archive.fosdem.org/2019/schedule/event/postgresql_fsync/) Tomas Vondra, FOSDEM'19

   • [Can Applications Recover from fsync Failures?](https://www.usenix.org/conference/atc20/presentation/rebello)

Q. How Does a Database Provide Crash Consistency Through Power Loss?<br>
A. The Write Ahead Log (WAL) - the crucial building block for ensuring atomic changes<br>

To continue: https://youtu.be/ehYcCTHRyFs?t=1325

---
---

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
