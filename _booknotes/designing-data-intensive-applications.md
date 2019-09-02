---
title: "Designing Data-Intensive Applications"
size: 1px
---

Important points and key learning gathered after reading Designing Data-Intensive Applications. Sorted and summarized by parts and chapters.

Still reading...

Databases
- Bitcask
    - Defaul storage engine in Riak
- SSTable (String sorted table)
- LSM (Log-Structured Merge-Tree)
    - Keeping a cascade of SSTables that are merged in the background
    - LevelDB, RocksDB, CockroachDB
- B-trees
    - Instead of variable sized segments, break databases into fixed-size blocks or pages
    - Read or write one page at a time
    - Leaf pages contain the value or contains reference to the page where value can be found
    - Always depth of O(log n)
    - For resiliency, WAL (Write ahead logs) or redo log is important
- Rule of thumb: LSM-trees are faster for writes
- Rule of thumb: B-trees are faster for reads
- Issues such as write amiplification are more pronouned in B-trees
- If write throughput is high and compaction cannot keep up with incomgin writes, the number of unmerged segments on disk keeps growing until you run of of disk space and reads also slow down because you need to check more segment files. To avoid this, have explicit monitoring and potentially throttle rate of incoming writes
- Indexes
    - Sometimes, actual rows are stored in heap files. Avoids duplicating data with multiple secondary indexes
    - Heap files might have forwarding pointers if there isn’t enough space and file had to be written elsewhere
    - Sometimes, this extra hop to a heap file is an overhead and in that case, it is desirable to store the indexed row directly within an index. This is know as clustered index. For ex. in MySQL InnoDB, primary key is always clustered index and secondary indexes refer to the primary key
    - Compromise between clustered and non-clustered index: covering index which stores some of a table’s columns within the index. Allows some queries to be answered by using the index alone (index covers the query)
- Transaction prcoessing or Analytics
    - OLTP (Online transaction processing) vs OLAP (Online analytic processing)
    - Data warehousing
    - Star and snowflakes schemas
        - Fact table with multiple dimension tables
        - Fact tables are huge
    - Column oriented storage
        - In data warehousing, we rarely do SELECT * queries. It’s usually particular columsn that we are interested in
        - Most OLTP dbs have storage laid out in row-oriented fashion: all values from one row of a table are stored next to each other. Row oriented storage engine loads all rows with 100s of attributes from disk into memory and filter out and parse the results. Takes long
        - column-oriented storage: instead of storing all values from one row together, store all values from each column together instead
        - If each column stored in separate file, a query needs to read and parse only those columns
        - Allows column compression using techniques such as bitmap encoding. Other techniques such as vectorized processing can also be implemented
        - A reow can be reassembled using the k-th value on each column
    - Data cubes and materialized views
        - Create cache on disk that computes an aggregate such as SUM. It needs to be updated every time underlying data is updated because it is a denormalized copy of the data. Virtual view is more like stored procedure that executes query on the fly
        - Special case of materialized view is data cube or OLAP cube. Basically, grid of aggregates grouped by different columns. Expensive to compute and rather rigid. Only used as performance boost for certain queries