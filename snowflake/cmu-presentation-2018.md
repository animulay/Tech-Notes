
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
• designed for small, fixed clusters<br>
• rely on complex ETL pipelines<br>
• typical usage pattern: nightly batch jobs<br>

Snowflake started around 2012 with cloud data warehouse vision:<br>
• data warehouse as a service with SQL frontend and no infrastructure to manage<br>
• multi-dimensional elasticity<br>
• on demand scalability<br>
• support for all types of data<br>

Shared-Nothing Architecture - Some Observations:
- Multi-cluster shared data architecture
- Tables are horizontally partitioned across nodes
- Every node has local storage
- Every node is only responsible for its local table partitions

#### Snowflake Architecture

#### References:
1. [slides](https://15721.courses.cs.cmu.edu/spring2018/slides/25-snowflake.pdf)
