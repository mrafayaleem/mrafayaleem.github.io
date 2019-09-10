---
title: "Designing Data-Intensive Applications (Part 2)"
---

Important points and key learning gathered after reading Designing Data-Intensive Applications. Sorted and summarized by parts and chapters.


**Chapter 5. Replication**

- **Sync vs asyc replication**
    - Sync is problematic if follower fails
    - With async, writer can continue
    - Keep one follower sync and rest async is also a good strategy in certain cases like tracking financial transactions
- **Setting up new followers**
    - Take snapshot. Should be associated to a specific position in DB. Also called binlog coordinates or log sequence number
    - Copy to new follower node instance
    - After copy, follower connects to master and processes backlog from the binlog coordinates onwards
    - We say that the follower has caught up
- **Handling node outages**
    - Follower failure: catch-up recovery by using it’s own logs
    - Leader failure: Do failover by promoting one of the nodes to leaders. Consider putting leader behind a load balancer to avoid reconfiguring clients
    - If async replication, new leader may not have received all writes. Most common to discard unreplicated writes at the expense of durability expecations
    - **IMPORTANT**
        - Discarding writes is especially dangerous if other storage systems outside of the database need to be coordinated with the database contents. For example, in one incident at GitHub, an out-of-date MySQL follower was promoted to leader. The database used an autoincrementing counter to assign primary keys to new rows, but because the new leader’s counter lagged behind the old leader’s, it reused some primary keys that were previously assigned by the old leader. These primary keys were also used in a Redis store, so the reuse of primary keys resulted in inconsistency between MySQL and Redis, which caused some private data to be disclosed to the wrong users.
    - In cases where two or more nodes think they are leaders, we have a split brain problem. Needs a mechanism to safely shut down one of the nodes
    - Also, timeouts should be set carefully before considering a leader dead
- **Implementation of replication logs**
    - **Statement based replication:** Leader logs every write as a statement (INSERT, UPDATE, DELETE) and that is forwarded to the followers
        - Nondeterministic functions like NOW() cause an issue
        - Statements using autoincrementing columns need to ensure that statements are executed in exactly the same order
        - Statements with side effects (eg. triggers) may result in different data occurring on each replica, unless they are absolutely deterministic
        - MySQL uses row-based replication by default now
    - **Write-ahead log (WAL) shipping:** Besides writing WAL to disk, send it to followers over the network as well. PostgreSQL uses that
        - Log describes data on very low level (pretty much at byte level) which ties it to the storage engine. If you change storage engine, not possible to run different versions of db software on leader and followers
    - **Logical (row-based) log replication:** Sequence of records describing writes to DB at the granularity of a row
        - For inserts, contains new values for all columns
        - For deletes, contains info to uniquely identify the row
        - For updates, contains info to uniquely identify row and the new values of all columns
        - Since decoupled from storage engine, more easy to keep backward compatibility allowing leader and follower to run different versions of DB software
        - Change data capture: Logical format is also easier for external applications to parse. Helfpul in cases where you want to send contents of DB to an external system such as data warehouse or build custom indexes and caches
    - **Trigger-based replication**
        - Allows more flexibility. For example, only want to replicate a subset of data or remove PII on replication
        - lets you register custom application code that is executed when a data change occurs in DB
        - Has greater overheads and more prone to bugs