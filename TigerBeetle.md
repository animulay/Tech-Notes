
## [TigerBeetle](https://tigerbeetle.com/)<br>

The Financial Transactions Database
---


YouTube presentation: [A New Era for Database Design with TigerBeetle](https://www.youtube.com/watch?v=ehYcCTHRyFs), InfoQ, Sept 21, 2023<br>
[Joran Greef](https://x.com/jorandirkgreef), Founder and CEO @TigerBeetle

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
