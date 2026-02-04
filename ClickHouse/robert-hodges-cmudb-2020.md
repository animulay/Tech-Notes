
YouTube video: [Introducing ClickHouse -- The Fastest Data Warehouse You've Never Heard Of (Robert Hodges, Altinity)](https://www.youtube.com/watch?v=fGG9dApIhDU) CMU Database Group, May 12, 2020

Robert Hodges, CEO www.altinity.com

ClickHouse features at a glance:
- Single C++ binary with extremely well written C++ code
- Shared-nothing architecture
- Column storage with vectorized query execution
- Built-in sharding and replication
- Apache 2.0 license

Bottom-up Design:

1. Specialized algorithms for common operations<br>
   data type: 14 GROUP BY algorithms<br>
   data size: whether data fits in memory<br>
   ordering: whether data is already [partly] sorted or not<br>
   data distribution: using multi-armed bandits to optimize LZ4 decompression<br>
                      there are multiple LZ4 algorithms

2. Vectorized query execution<br>
   requires support for SIMD SSE 4.2+<br>
   efficient dispatch on all available cores<br>

ClickHouse is optimized for fast data insertion.<br>
The data is reorganized later for efficient querying.<br>

You can query the data as soon as the insert is complete.<br>

Multi-master eventually consistent with Zookeeper for consensus (**prefers availability over consistency**).

The transactional unit in ClickHouse is the "**Part**".<br>
The individual parts are written atomically.<br>
There is no notion of doing row-level transactions. e.g. data deletion results in re-writing the parts.

ClickHouse materialized views are synchronous post-insert triggers.

common uses:<br>
. Aggregation<br>
. automatic reads from Kafka<br>
. pre-computing last-point queries<br>
  common use-case for time series data<br>
. changing sorting or primary key<br>

The data doesn't change much over time.

ClickHouse has great system tables. e.g. system.parts, system.columns<br>

**There's no query optimizer**.

ClickHouse has sharding and replication built-in.

shard : dis-joint piece of dataset<br>
        tables broken into datasets and spread across the nodes<br><br>
        tables are grouped and you can replicate between tables.

**ClickHouse doesn't have a cost based optimizer**.

Use-cases where ClickHouse shines:
- very large amounts of structured data<br>
  e.g. network flow logs (60 columns CDN public/private cloud)

- you have very large number of records and want to get results quickly

---

## Reference

- [The Secrets of ClickHouse Performance Optimizations at BDTC 2019](https://www.youtube.com/watch?v=ZOZQCQEtrz8), Jan 7, 2020
- [Evolution of data structures in Yandex.Metrica](https://highscalability.com/evolution-of-data-structures-in-yandexmetrica/), Alexey Milovidov, 18 Sep 2017
