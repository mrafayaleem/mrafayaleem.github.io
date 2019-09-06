---
title: "High Performance MySQL"
---

Important points and key learning gathered while reading High Performance MySQL.

Chapter 5. Indexing for High Performance
- If you index more than one column, column order is very important
- It can most efficently search on leftmost prefix of the index
- Ex key(last_name, first_name, dob)
    - Sorts according the the order of columns given. In this case, by last_name, first_name and finally dob
    - Works on:
        - Full key value with all columns
        - Match on last_name (left most prefix)
        - Match on last_name starting with J
        - Last names between Allen and Barymore
        - Last name Allen and first_name starts with K
        - B-Tree normally support index-only queries that access only index, not row storage
        - Can be used of lookups and sorting such as order by because sorted
    - Doesn’t work
        - Last name ends with a particular letter
        - Only using first_name or dob
        - Can’t skip columns like last_name and dob but skipping first_name
        - Can’t optimize any columns to the right of the first range condition. Ex. `WHERE last_name="Smith" AND first_name LIKE 'J%' AND dob='1976-12-23'` 
    - Create indexes with same columns in different orders to satisfy different queries
- Hash indexes (pg 152)
    - Specialized cases
    - Implementing own pseudohash is blazing fast because the index is on much smaller value. Imagine index on url vs CRC32(url)
    - To avoid collisions, always use a WHERE clause along with your index in the pseudohash table
    - Can automate generation of these hashes by setting up triggers
- R-Tree indexes don’t require to operate on left most indexes. Not well supported in MySQL
- Good column order
    - ROT: Place most selective columns in the index first
- Clustered Indexes
    - Rows stored in index’s leaf pages
    - Can only use one clustered index per table because can’t store the rows in two places at once
    - InnoDB clusters data by primary key
    - You can keep related data closer, for ex cluster on user_id to retrieve all user’s messages by fetching only a few pages
    - Data access is fast because both index and data are kept together
    - Queries using covering indexes can use PK values contained at the leaf node
- Covering indexes
    - Secondary indexes hold the row’s PK values at their leaf nodes. Thus a secondary index that covers a query avoids another index lookup in the primary key
    - `EXPLAIN SELECT * FROM products WHERE actor='SEAN CARREY' -> AND title like '%APOLLO%'\G`
    - No index covers this query because all columns are selected and no index covers all columns. My understanding is that if only an indexed column was selected and given that actor is also indexed, engine would fetch PK using actor and then fetch the column in the SELECT clause. This way, it won’t read the whole data and the index will cover the query 
