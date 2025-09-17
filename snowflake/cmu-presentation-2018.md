
[CMU Advanced Database Systems - 25 Ashish Motivala [Snowflake] (Spring 2018)](https://www.youtube.com/watch?v=dABd7JQz0A8)<br>
CMU Database Group<br>
May 8, 2018

Presenters:
- Ashish Motivala, prior work on in-memory database @ Oracle
- Jiaqi Yan, prior work on query optimizer @ Oracle

[Snowflake](https://www.snowflake.com/en/) is
- an enterprise data warehouse product for Online Analytical Processing (OLAP)
- multi-tenant transactional database in the cloud

Founders:
- Benoît Dageville, data architect @ oracle
- Thierry Cruanes, data architect @ oracle
- Marcin Żukowski, co-founder of Vectorwise

Pre-cloud traditional data warehouse (DW)<br>
• were designed for small, fixed clusters<br>
• relied on complex ETL pipelines<br>
• typical usage pattern: nightly batch jobs<br>

Snowflake started around 2012 with vision:<br>
• cloud data warehouse as a service with SQL frontend and no infrastructure to manage<br>
• multi-dimensional elasticity<br>
• on demand scalability<br>
• support for all types of data<br>

Some observations about typical Shared-Nothing Architecture:
- horizontal partitioning of tables across the nodes
- every node has local storage
- every node is only responsible for its local table partitions

Some of the issues with Shared-Nothing Architecture:<br>
• couples compute and storage<br>
• resizing compute requires redistributing lots of data<br>
• cannot shutdown unused compute<br>
• failures, upgrades may cause downtime and results in performance penalty<br>

### Snowflake Architecture
- Multi-cluster, *Shared data* Architecture broadly split into 3 layers
  1. Cloud Services
  2. Virtual Warehouse(s)
  3. Data Storage

![Multi-cluster, Shared data Architecture](./snowflake-shared-data-arch.png)

- Storage decoupled from compute
- Can handle any data (structured as well as semi-structured)
- Unlimited scalability
- Instant cloing
- Highly Available
  - Durability: 11 9's
  - Availability: 4 9's
- Low cost compute on demand

Snowflake<br>
• has a concept of a virtual data warehouse (DW).<br>At a time, there can be multiple virtual DWs in action each working on its own data.<br>
• has connectors for other non-SQL solutions like Apache Spark<br>
• has dashboarding features like Tableau, Qlik<br>
• Data in S3 or block storage is immutable<br>
• Large metadata associated with data e.g. accounts, users, sessions, billing, SQL statement, transactional changes<br>
• The metadata storage is a separate system, available to all the services in the DW<br>
• Cloning:<br>
  - Each virtual DW gets its own copy of the subset of data that's needed.<br>This offers really good cache locality.<br>

#### Cloud Services - manage "everything other than query execution"
• Authentication & Access Control<br>
• Infrastructure Manager<br>
• Optimizer<br>
• Transaction Manager<br>
• Security<br>
• Metadata<br>

a virtual DW
. completely stateless worker-set
. work on a plan
. fetch data from S3
. cache data locally
. poll for more work when done

[slide12] Data Storage Layer
. stores table data & query results
  immutable micro-partitions
  turned out to be a huge blessing

  <<S3 (object storage) was built for write-once-read-many (WORM) use-case>>
. micro-partitions
. S3 performance not that good; but addressed via local caching

[slide13] Table Files
. immutable 16 MB micro-partitions
. what a micro-partition looks like
. PAX format - hybrid columnar storage
. each column is stored separately within the same file
  compressed data within each column
  compression scheme depending on the data type
  numbers => base encoding
  strings => trie's & dictionary encoding
  JSON    => trie
. header maintains an offet for each column within the file
  fseek()



#### References:
1. [slides](https://15721.courses.cs.cmu.edu/spring2018/slides/25-snowflake.pdf)
