# ```Part 1: The ACID PRINCIPLE```
## ```Transaction```
-	Multiple queries treated as a single unit of work.
-	Transaction lifecycle consists of 4 parts internally: BEGIN, COMMIT 
(Store in memory/RAM), WRITE (Store in disk).
-	However, these can differ by DB engines, each DB engine has its own customized and 
complex transaction management system. Some focus more on commits some focus on crashes 
while some try to balance.
-	For COMMIT & WRITE there can be two approaches basically.
-	Each smaller unit is committed and written in disk -> needs lots of I/O, very fast, 
hard to rollback, used by Postgres
-	Every commit is prepared first then all are stored in disk -> slower, less IO, low 
failure chance, easier rollback
-	```But, what if the DB crash during a commit?```
-	Read only transaction is also normal, because it gives snapshot of a moment, without
transaction, in regular reads, things may get changed in the middle.

**Example of a transaction**

![img_1.png](resources/img_1.png)


## `Atomicity`
Let’s say for the first example, we’ve deducted from the first account and then the 
DB crashed! The balance is never added in second account!
**Def: A block of queries must be executed entirely or none at all.**

## `Isolation`
Operations must be separated from all the outside-block actions. Traditional 
isolation related problems include ```Non-repeatable read``` 
(in a single block there are two read queries on the same table, after the first query 
another query from another thread updates data and commits so the second query reads 
different!), ```Dirty read``` (similar to non-repeatable but in this one, the second thread 
changes mind and doesn’t commit changes & rolls back after first thread’s second query),
```Phantom read``` (in this case, instead of updating, a new row is inserted by other thread. 
As it is completely new and first thread has no idea of it, so it is the hardest to fix),
```Lost update``` (the most common concurrency issue, two thread reads data, thread 1 updates it,
thread 2 updates it, the update of thread 1 is overwritten.)

**Example of dirty read:**

![img_2.png](resources/img_2.png)

Different database uses different isolation levels by default to prevent different 
reading phenomena.

_Table: Isolation levels vs read phenomena_

| Isolation level | Dirty reads  | Lost updates | Non-repeatable reads | Phantoms |
|-----------------|--------------|--------------|----------------------|----------|
|Read Uncommitted| May occur    |May occur|May occur|May occur|
|Read committed| Don’t occur  |May occur|May occur|May occur|
|Repeatable read| Don’t occur  |Don’t occur|Don’t occur|May occur|
|Serializable| Don’t occur  |Don’t occur|Don’t occur|Don’t occur|

```Read uncommitted``` – No isolation, ```Read 
committed``` – a transaction only sees others’ committed transaction,
```Repeatable read``` – while a transaction is running, 
its usages rows will not be changed, kind of locking, 
```Serializable``` – disables concurrency,
```Snapshot``` – takes a snapshot of entire context and uses it during whole transaction.

Also, for locking, there are two types of locks. Pessimistic locks 
the context (row, table, page), Optimistic (doesn’t lock at all, triggers 
error/fail for a transaction when any anomaly happens.

## ```Consistency```
If a data is updated, all other related (other table, foreign key, 
other lined database etc.) and dependent data shall get the update immediately.

## ```Durability```
Database must store data in disk and shouldn’t lie without actually saving. As a result, DB engines ignores OS cache and stores data in disk by its own using f-sync commands.


# ```Part 2: Storing data```
A ```row``` is a single record, a `column` is a set of single attribute values, 
a `row_id` field is used by default in many databases which actually behaves 
like primary column internally, a `page` is a logical block with fixed size 
(8KB in Postgres, 16KB in MySQL) consists of multiple rows/columns, an `IO` is a read/write
operation, it reads one or more page at once, cannot read single row/column. IO is expensive. 
IO reads from OS Cache too, not always from disk. Optimizing an IO is DB engines’ 
one of the major goals, while reducing its use is programmers’ major goal. 
Tables are stored in `Heap` DS, traversing Heap is expensive as it gives you all 
pages and requires iterating through everything to find a row. Index is another 
DS which keeps pointers to heap using B-Tree. Example of reading from index & heap 
using two IO (imagine finding last row without index!)
![img_3.png](resources/img_3.png)

Data storing can be both row and `column based`, the above one is example of row based,
however in column-based storing, only a limited (equals to number of columns) number of rows are created, each row contains all the values for a particular column. Example:
![img_4.png](resources/img_4.png)
![img_5.png](resources/img_5.png)

Good for aggregation, good for compression (as multiple record has same value so 
stored as value as a key with list of record id), very problematic in queries especially 
in select * queries!

###  Index oriented tables
An `index-oriented` table or a `clustered table` is the one which’s rows are 
organized based on the primary key, meaning that the row with primary key 
N+1 will always be after row N and before N+2. Fast but space-costly! i.e.: 
row with PK 1 and 3 are inserted, an empty row for PK 2 will be reserved. 
It’s by-default in MySQL, Postgres doesn’t have it (as it has a default PK internally 
and all user defined PK is actually SK), other DB allows to create it.

# `Part 3: Hands on with indexing`
Time to practice some indexing related queries!


```bash
# Run a Postgres docker container with custom password, custom name and latest docker version.
docker run -e POSTGRES_PASSWORD=postgres --name pg1 postgres;

# Log into Postgres in terminal
docker exec -it pg1 psql -U postgres;
```

```bash
# Create table emp
create table emp(id serial primary key, age int, salary int, experience int, rank int);

# Insert 10 million rows
insert into emp(age, salary, experience, rank) select random() * 50 + 18, 
random() * 190000 + 10000, random() * 20, random() * 9 + 1 from generate_series(0, 10000000);

# See table detail, total or rows 
\d emp; select * from emp limit 100; select count(*) from emp;
```


```bash
# Because id is PK and has indexing, this query took only 4.2ms in first run 
# and for any subsequent run it only took 0.32ms because of caching
explain analyze select * from emp where id = 1000222;

# Salary column doesn’t have index, it took 640ms without cache and 120ms with cache.
explain analyze select * from emp where salary = 51240;

# Now lets create index on salary field
create index emp_salary on emp(salary);

# The same previous query now took 9.6ms without cache and 0.91ms with cache!
explain analyze select * from emp where salary = 51240;

# Takes 600ms & 140ms with & without cache, because though salary is indexed but needed 
# to check all salary here.
explain analyze select * from emp where salary > 190000 and experience < 2 and age < 21;
# Note: If we change the condition to salary = something then query becomes extremely fast 
# because of indexing as expected. However, if we set salary condition in a way that it can 
# vary within a range then still the DB utilizes index!

```

### More on Indices
**`A scenario:`** If a table has columns a, b, c and we have index(a, b) then:
-	Query with ‘a or b’: parallel seq scan – 167ms -> same as without index
-	Query with ‘a and b’: index scan – 4ms -> fast because of index
-	Query with ‘b’: parallel seq scan – 176ms -> same as without index
-	Query with ‘a’: bitmap index scan – 6ms -> special case: as a column is first key in index, its fast!

There are different types of scans. `Index scan`, `Bitmap index scan`, `Table scan` 
(these names are based on PG, name can vary through DB’s however implements conceptually 
same thing). `Sequential scan or table scan` is the brute force, traditional, 
top to bottom iteration of pages and rows in a table. Parallel IO is used to make the 
scan faster, it is O(N) in terms of time complexity. `Index scan` is the faster one with 
O(logN) time complexity as it searches in the index tree, on the other side the 
`Bitmap index scan` is somewhere between the above two. Though index search 
is fast but jumping between index and the heap memory is costly as it requires 
IO, as a result, for a ranged query where there can be a lot of results the
DB engine first operates index scan and prepares a bit map for pages. For example –

| Page1 | Page2  | Page3 | Page4 | Page5 |
|-|-|-|-|-|
|0|0|1|0|1|0|

Of course, the actual table may be more complex with additional information but the 
minimal one shown here means that we need to scan page 3 and page 5 to prepare query 
result. So, it does index scan -> prepares bitmap then jumps to heap for once only.
Time complexity is same as index scan but this one is space costly because of the bitmap table.

**```Bloom filter```**: In cases where we require very simple yet extremely repetitive 
query such as whether an email or username exists or not, a hash-based elimination 
probability analyzer algorithm called bloom filter comes in! Using multiple hash functions 
and a bit array, it quickly and space-efficiently checks a probability of an items existence; 
query is executed only after it says the data may exist.

# `Part 4: Indexing data structures`
## B Tree 

![img_6.png](resources/img_6.png)

### `Properties`
- All leaves are at the same level.
- B-Tree is defined by the term minimum degree t. The value of t depends upon disk block size.
- Every node except the root must contain at least t-1 keys.
- All nodes (including root) may contain at most (2*t – 1) keys.
- Number of children of a node is equal to the number of keys in it plus 1.
- All keys of a node are sorted in increasing order. The child between two keys k1 
and k2 contains all keys in the range from k1 and k2.
- B-Tree grows and shrinks from the root which is unlike Binary Search Tree. 
Binary Search Trees grow downward and also shrink from downward.
- Like other balanced Binary Search Trees, the time complexity to search, insert, 
and delete is O (log n).
- Insertion of a Node in B-Tree happens only at Leaf Node.
- Used as default in PostgreSQL 	Everything is  same as B Tree except for the 
value storing mechanism. In B+ tree the intermediate nodes doesn’t store any value, only keeps the keys and the leaf nodes stores values, additionaly the leaf nodes keeps a pointer  to the next node which brings lot of facilites wich such little cost.
- Used as default in MySQL, Oracle and most other DB systems. 

### `Search Efficiency`	
Good, as it maintains a balanced structure for efficient searching	Better, 
as all values are in leaf nodes, making range queries faster

### `Insertions/Deletions`
Efficient, with minimal re-balancing needed due to its balanced nature 
Efficient, similar to B-trees, but may involve more re-balancing

### `Space Utilization`
Can be less space-efficient as internal nodes store data
More space-efficient as only leaf nodes store data

### `Range Queries`
Less efficient, as search may involve traversing internal nodes	More efficient, 
as all data is at the leaf level, enabling faster range queries

### `Node Structure`	
Internal nodes store both keys and values, potentially making them larger	
Internal nodes store only keys, making them smaller and faster to navigate

### `Traversal`	
May require more traversals due to internal nodes storing data	More straightforward, 
as traversal is often limited to leaves for data retrieval
Indexing	Suitable for general indexing needs, including key-value pairs	
Ideal for indexing where range queries and sorted data access are common

## B++ Tree

![img_7.png](resources/img_7.png)

### `Properties`
Everything is  same as B Tree except for the value storing mechanism. In B+ tree the intermediate nodes doesn’t store any value, only keeps the keys and the leaf nodes stores values, additionaly the leaf nodes keeps a pointer  to the next node which brings lot of facilites wich such little cost.
Used as default in MySQL, Oracle and most other DB systems.

### `Search Efficiency`
Better, as all values are in leaf nodes, making range queries faster

### `Insertions/Deletions`
Efficient, similar to B-trees, but may involve more rebalancing

### `Space Utilization`
More space-efficient as only leaf nodes store data

### `Range Queries`
More efficient, as all data is at the leaf level, enabling faster range queries

### `Node Structure`
Internal nodes store only keys, making them smaller and faster to navigate

### `Traversal`
More straightforward, as traversal is often limited to leaves for data retrieval

### `Indexing`	
Ideal for indexing where range queries and sorted data access are common

    Code: Implement indexing & connection pulling using spring and hibernate: https://github.com/feehaam/Postgres_indexing_with_spring_jpa

# `Part 5: Distribution`
### `Partitioning`
Partitioning is `storing` rows (horizontal) or columns (vertical) `in multiple tables 
based on some logic.` The most common types of partitioning are `ranged` partitioning, 
`list` partitioning, `hash` partitioning, note that all these can be classified as horizontal
partitioning. Assume there is a many to many post-reaction table with columns 
(userId, postId, i reactionType, reactionTime), even doing index on userId or postId will 
not work because there can be billions of users and trillions of posts! So, what we 
can do is break the table based on id range! There will be multiple tables with same 
data each table containing a limited rage amount, for example the first table may have 
reactions with postId 1-10,000,000 next one with next 10 million then 10 and so on. 
Partitioning makes tables & queries `lighter`, `faster`, `complex`.

    Code: Implement paritioning using Java/Spring and PostgreSQL: https://github.com/feehaam/Postgres_partitioning_with_spring_hibernate_jdbc


### `Sharding`
Sharding is a different concept of distributing the data in `different databases`. 
It is needed `when there are so much data that the DB space is struggling`. It is very 
similar to partition but only in different server. Sharding can be employed by dividing 
the data based on a sharding key, such as user ID or geographical region. Each shard 
contains a subset of the data and operates independently, allowing the system to 
handle increased traffic and data volume by adding more servers as needed. This 
approach alleviates the performance `bottlenecks of a single server` and facilitates 
`horizontal scaling`. It also can offer `additional security` based on sharding key. 
Transaction, schema changes, joins, rollbacks is a real headache because of multiple databases.

    Code: Implement sharding across different databases using Java/Spring and PostgreSQL: https://github.com/feehaam/Sharding_across_multi-DB_with_spring_postgres_mySql

### `Replication`
Replication is a technique used to enhance the availability and 
reliability of a database by creating and maintaining `multiple copies` of the same data 
across different servers. In replication, data changes made to a `primary` database 
(often called the `master`) are copied to one or more `secondary` databases 
(known as replicas or `slaves`). This ensures that if the primary database fails, 
the secondary copies can take over, `minimizing downtime and data loss`. For example, in a 
high-traffic e-commerce site, replication can be employed to distribute read queries 
among several replicas, thereby reducing the load on the primary server and 
improving overall performance. Replication also supports disaster recovery 
and backup strategies by keeping up-to-date copies of data in different locations. 
While it `introduces complexity in` managing data `consistency` and `synchronization` 
across multiple servers, replication is crucial for maintaining high availability 
and scaling read operations in critical applications.


# `Part 6: Concurrency`
### `Shared lock & exclusive lock`
When `shared lock or read lock` applied on a/some row then it means that we're reading the rows 
and somebody `must not change or update` things in the middle of the way, whoever others can 
read and make more shared lock too, but while active no one can acquire an exclusive lock. 
`Exclusive lock or write lock` on the other hand doesn't allow others to both read and write, it
locks entirely because it means that I'm writing something everybody should read after I'm done
otherwise they will get some updated and some old values.

### `Deadlock`
Deadlock occurs when one thread(a) locks a part(a) and tries to access another part(b) but can't
because that part is locked by another thread(b) and that thread tries to access(a) and can't
as it is locked by thread(a), so thread(a) can't fining because of thread(b)'s lock and 
thread(b) also can't finish because of thread(a).

However in databases, waiting is allowed while locked, but the moment when deadlock starts
the thread that goes to deadlock second is immediately failed. 

In the below example, the column id is unique, first transaction 1 adds an entry 10 then 
transaction 2 adds 20, then right when transaction 1 tries to add 20 it starts waiting because
transaction 2 is working on the row that has id 20 and transaction 2 is still processing. 
But right when transaction 2 tries to add 10, it also starts waiting for T1 because it is working
or row 10 and still processing because waiting for T2 who is waiting for T1 who is waiting for T2
who is waiting for T1..... DEADLOCK! 

Then the DB engine terminates the T2 as it entered deadlock last so all of its changes are 
rolled back and see in the select query, both row is shown as added by T1. Note that T1 executes
right when T2 ends doesn't matter it ends normally or by rollback or by failure.

![img_10.png](resources/img_10.png)

### `The two phase problem`
This is the problem where there are 2 or N same steps being executed by two threads in an 
ordered manner as a result the thread with last action wins. For example, lets say thread T1 and
thread T2 clicks to book two ticket at the same time. Both refresh same time so `ticket no 2324` 
is shown available (booked_by = null in below screenshot). Then both wants to book that ticket
T1 books it and ends the transaction then T2 books it and ends the transaction. Boom! T1's 
action is gone! See the below steps in screenshot.

![img_13.png](resources/img_13.png)

Now as a solution to this, we can apply an exclusive lock on `ticket no 2325` from each thread, 
as shown in the screenshot, the second thread is not getting any result as thread 1's 
transaction is in progress, after thread 1 is done, thread two will execute the select query 
and notice that booked_by has a value. And won't attempt to book it.

![img_14.png](resources/img_14.png)

Bellow, though both transactions started at the same time and both T1 and T2 got isBooked = 0 
then why the second update failed? 
Because, Postgres by default applies `read commited` and before commiting a transaction it reads values
again then it found isBooked = 1. This result may differ in different DB engines based on their implementations.

![img_15.png](resources/img_15.png)

# `Part 7: Cursors`
While a ranged amount of data needed to be processed, the whole operation can be quite lengthy
and memory consuming, then server gets all. `Cursor` is the solution by which we can 
`create a stream of data`(Like future in Java), in is ready to be executed, 
`and we can get records chunk by chunk` from DB storage
using fetch query. It is quite useful way of sorting a table with huge size data because traditional 
sorting (Though DB engines do good RAM management still) may use too much RAM. There are two 
types of cursor: `Server side cursor` (Pros: handled by DB, better for large data, small network 
traffic, less client RAM, consistent data. Cons: complex, client stays in hold) & `Client side 
cursor` (Pros: By default this is what we do so no complexity, reduced server load, consistent data 
because of immediate data access. Cons: huge client memory, huge network traffic)

# `Interview questions`

## `Basic`
#### `Q` When would you consider a NoSQL design over an SQL design?
`A` I consider NoSQL design when the application requires high scalability, flexibility to handle unstructured or semi-structured data, and rapid development cycles where schema changes are frequent and varied.

#### `Q` What do you understand about BCNF?
`A` BCNF refers to the Boyce Codd Normal Form, an advanced version of the third normal form that does not feature overlapping candidate keys.

#### `Q` What do you understand by database testing?
`A` a) Testing of data integrity and validity. b) Performance of database. c) Testing of procedure, functions, and triggers

### `Q` What is denormalization in database design?
`A` Denormalization is the process of adding redundancy to a database to improve read performance, typically at the cost of write performance and increased storage requirements. It involves combining tables or adding redundant data to reduce the need for complex joins.

### `Q` What is query pagination and why is it useful?
`A` Query pagination involves splitting large datasets into smaller, manageable chunks or "pages" that can be individually retrieved and displayed. It's particularly useful in web applications for enhancing user experience and reducing server load by only fetching data as needed.

### `Q` Explain the difference between DELETE and TRUNCATE commands.
`A` DELETE removes rows from a table based on the condition provided and can be rolled back. TRUNCATE removes all rows from a table without logging each row delete, thus it cannot be rolled back in most RDBMS. It is faster and uses fewer system and transaction log resources.

### `Q` Describe a scenario where an index should NOT be used.
`A` Indexes should not be used in tables with frequent large batch updates or high transaction volumes where the overhead of maintaining the index might outweigh the performance benefits. Also, for very small tables, the performance benefit might be negligible.

### `Q` What is a non-correlated subquery?
`A` A non-correlated subquery is a subquery that can be run independently of the outer query and returns a value or set of values to be used by the outer query. This type of subquery does not depend on data from the outer query to execute.

### `Q` How can databases handle concurrency?
`A` Databases handle concurrency through locking mechanisms (optimistic and pessimistic locking), versioning, and isolation levels, which ensure data consistency and integrity when multiple users access data at the same time.

### `Q` What is the role of a DBA (Database Administrator)?
`A` A DBA is responsible for installing, configuring, upgrading, administering, monitoring, and maintaining the security of databases in an organization. They ensure data availability, performance, and compliance with industry regulations.

### `Q` What are stored procedures and how are they utilized?
`A` Stored procedures are precompiled sets of SQL queries that are stored in the database. They can be invoked/reused by applications and can help secure databases by encapsulating business logic within the database, reducing overhead and enhancing security by limiting direct SQL operations.

### `Q` What is SQL ANSI standard?
`A` SQL ANSI standard refers to the American National Standards Institute (ANSI) standards for SQL, which provide guidelines and specifications to ensure SQL language syntax and operations are consistent across different database systems, promoting interoperability and reducing the complexities involved in managing multiple database systems.

#### `Q` What is the N+1 selects problem in ORMs?
`A` The N+1 selects problem occurs when an ORM fetches objects and their associations with one query for the master record and then one query for each related detail record, leading to many unnecessary database hits. This can be mitigated by using eager loading techniques.

#### `Q` What is SQL injection and how can it be prevented?
`A` SQL injection is a security vulnerability through which an attacker can execute arbitrary SQL code on a database by manipulating the inputs to a web application. It can be prevented by using parameterized queries, prepared statements, and proper input validation.

#### `Q` Explain the difference between INNER JOIN and OUTER JOIN.
`A` An INNER JOIN returns only the records where there is a match in both joined tables. An OUTER JOIN (LEFT, RIGHT, or FULL) returns all records from one side of the join and the matched records from the other side; if there is no match, NULL values are used to fill in the gaps.

#### `Q` How do database views work and what are their advantages?
`A` A database view is a virtual table based on a SQL query which pulls data from underlying tables. They simplify complex queries, enhance security by restricting access to underlying data, and can aid performance by serving as a mechanism for SQL optimization.

#### `Q` What are materialized views and how do they differ from standard views?
`A` Materialized views are database objects that contain the results of a query and can be refreshed on demand; they physically store the data. In contrast, standard views are definitions of queries that dynamically retrieve data when used but do not store the data themselves.

#### `Q` What is query pagination and why is it useful?
`A` Query pagination involves splitting large datasets into smaller, manageable chunks or "pages" that can be individually retrieved and displayed. It's particularly useful in web applications for enhancing user experience and reducing server load by only fetching data as needed.






## `Atomicity`
#### `Q` How does a database manage atomicity in case of system failures?
`A` Databases manage atomicity using transaction logs. If a system fails in the middle of a transaction, the database can use the log to roll back any partial changes made by the transaction to ensure it is either fully completed or fully undone.

#### `Q` What are transaction logs, and how are they related to atomicity?
`A` Transaction logs are records of all changes made during transaction processing. They are crucial for atomicity as they allow the database to roll back changes to maintain consistency if a transaction fails.

#### `Q` Is it possible to achieve atomicity across multiple databases? How?
`A` Yes, atomicity across multiple databases can be achieved through distributed transaction protocols such as the two-phase commit or three-phase commit protocols.

#### `Q` How do microservices architectures handle atomicity?
`A` In microservices architectures, atomicity is often handled using distributed transactions, compensating transactions, or relying on eventual consistency and domain-driven design patterns to ensure logically atomic operations.

#### `Q` What is the two-phase commit protocol and how does it relate to atomicity?
`A` The two-phase commit protocol is a method of ensuring atomicity across distributed systems or databases. It ensures that all parts of the transaction commit only if all involved parties agree, or all rollback if any party disagrees.

#### `Q` How does an atomic transaction impact database performance?
`A` While atomic transactions ensure data consistency and integrity, they may lead to performance overhead due to the need for locking resources and logging to manage the rollback process.

#### `Q` Can you explain the importance of rollback in maintaining atomicity?
`A` Rollback is crucial for maintaining atomicity as it allows the database to undo any operations of a transaction that cannot be completed fully, thus ensuring that no partial transactions are committed.

#### `Q` How do atomicity and error handling relate in DBMS?
`A` Atomicity ensures that any errors occurring during a transaction do not affect the database state by guaranteeing the transaction is either fully committed if successful or fully rolled back in the case of an error, maintaining data integrity and consistency.






## `Consistency`
#### `Q` Define consistency in the context of databases.
`A` Consistency in databases refers to ensuring that all data follows defined rules and constraints, such as database schema, 
and foreign key constraints. It ensures the data is accurate and reliable post any transaction and all related data is synced.

#### `Q` What mechanisms do databases implement to ensure consistency?
`A` Databases use constraints (primary key, foreign key, not null, unique), triggers, and transaction controls to maintain consistency.

#### `Q` How does a database enforce foreign key constraints to maintain consistency?
`A` Databases enforce foreign key constraints by preventing operations that would lead to invalid data references, such as deletions or modifications that violate the integrity of the links between tables.

#### `Q` Provide an example of a situation where database consistency might be at risk and how it is protected.
`A` During simultaneous updates on the same data by multiple transactions, database locks are used to serialize these transactions, ensuring that each transaction's changes are isolated from others, thereby maintaining consistency.

#### `Q` Explain the difference between transaction consistency and database consistency.
`A` Transaction consistency refers to maintaining a proper state during the execution of a single transaction, whereas database consistency refers to the correctness and adherence to constraints across the entire database.

#### `Q` How do decentralized databases maintain consistency?
`A` Decentralized databases maintain consistency using consensus algorithms like Paxos or Raft, which ensure that all nodes in the system agree on the current state of the data, despite the possibility of node failures or network partitions.

#### `Q` State CAP theorem.
`A` The consistency & availability of data can not be ensured at once, guarantee of one will lead to lack of another. 





## `Isolation`
#### `Q` What is the role of isolation in databases?
`A` Isolation ensures that transactions are securely and independently processed without interference, even when multiple transactions occur concurrently.

#### `Q` Describe a scenario where poor isolation levels can affect a system's performance or integrity.
`A` Lower levels of isolation, like Read Committed, can allow phenomena such as non-repeatable reads or phantom reads, affecting the accuracy of data reporting and analysis.

#### `Q` How do isolation levels affect database performance and concurrency?
`A` Higher isolation levels, like Serializable, prevent concurrency errors but can lead to decreased system performance due to increased locking and blocking. Lower levels increase performance but at the risk of introducing anomalies.

#### `Q` What are phantom reads and which isolation level prevents them?
`A` Phantom reads occur when a transaction reads a row in a table, and a subsequent read within the same transaction sees rows that were inserted/deleted by another transaction. Serializable level prevents them.

#### `Q` Explain the concept of "dirty read" and how it is controlled.
`A` A dirty read occurs when a transaction reads data another transaction has not yet committed, potentially leading to data inaccuracies. It's controlled by setting an appropriate isolation level, such as Read Committed.

#### `Q` What is snapshot isolation, and how does it work?
`A` Snapshot isolation allows a transaction to operate on a consistent snapshot of the database, taken at the start of the transaction, to avoid being affected by concurrent modifications, thereby reducing locking needs and improving concurrency.

#### `Q` Explain the difference between read committed and serializable isolation levels.
`A` Read committed ensures that only committed data is read, preventing dirty reads, whereas serializable is the strictest level, preventing not only dirty reads but also non-repeatable reads and phantom reads by serializing the execution of transactions.

#### `Q` What are write skews, and under which isolation level can they occur?
`A` Write skews are a form of anomaly where two transactions read overlapping data, see no conflicts, and then update parts of the data leading to inconsistent results. They can occur under Snapshot Isolation or lower levels but not under Serializable.

#### `Q` What is lock escalation, and how does it affect database performance and isolation?
`A` Lock escalation is the process of converting many fine-grain locks (like row-level locks) into fewer coarse-grain locks (like table-level locks). While this can reduce memory overhead and improve performance, it might decrease concurrency and isolation.

#### `Q` How can one prevent deadlocks in a database system?
`A` Preventing deadlocks can be managed by ensuring transactions acquire locks in a consistent order, implementing timeout policies where transactions automatically rollback if they cannot obtain a lock, or using deadlock detection algorithms that rollback one of the transactions involved.

#### `Q` Explain how database replication factors into maintaining isolation.
`A` Replication can complicate maintaining isolation as the same data might be read and written concurrently across multiple copies. Ensuring that all replicas are synchronized and that write operations are propagated correctly is essential to maintain isolation.







## `Durability`
#### `Q` What does durability mean in a database system?
`A` Durability guarantees that once a transaction has been committed, it remains so, even in the event of a crash, power loss, or other system failures.

#### `Q` How do database systems ensure data durability?
`A` Database systems use transaction logs that record all changes. These logs are used to recover committed transactions if the database crashes, thus ensuring durability.

#### `Q` What role do non-volatile memory storages play in ensuring data durability?
`A` Non-volatile memory, such as SSDs or magnetic disks, ensures that data is not lost when the device is powered off, maintaining the durability of the data stored on them.

#### `Q` How does a database handle durability during a system upgrade?
`A` During system upgrades, databases ensure durability by performing all pending transactions, backing up data, and maintaining comprehensive logs that can be used for recovery if an update failure occurs.

#### `Q` Can durability be compromised, and how can one mitigate such risks?
`A` Durability can be compromised in scenarios involving disk failures or data corruption. Regular backups, RAID configurations, and redundant system setups can mitigate such risks, ensuring data is not lost permanently.

#### `Q` How do databases use write-ahead logging (WAL) to ensure durability?
`A` Write-ahead logging involves writing changes to a log before the actual data pages are written to disk. This ensures that in case of a crash, the database can recover by replaying the log entries, preserving the durability of transactions.

#### `Q` What is the role of the redo log in a database system?
`A` The redo log records all changes made to the database as part of the transaction log. It is used during recovery operations to redo, or reapply, all actions of transactions that had committed but whose changes had not yet been fully written to disk.

#### `Q` How does a database's cache mechanism impact durability?
`A` While caching improves performance by reducing disk I/O, it can pose risks to durability if not properly managed, since data in cache is usually not written to disk immediately. Ensuring that cached data is periodically flushed to disk and using transaction logs can mitigate these risks.

#### `Q` Describe the trade-offs involved in improving database durability versus enhancing performance.
`A` Improving durability often involves additional disk I/O operations, logging, replication, and other resource-intensive activities that can detract from performance. Balancing these aspects involves optimizing how data is logged and replicated, and choosing the right hardware and technology strategies to meet both durability and performance needs effectively.







## `Transaction`
#### `Q` How does Spring Boot support transactions? Describe how you would configure and use transactions in a Spring Boot application.?
`A` Spring Boot supports transactions through the @Transactional annotation. `@Transactional` can be applied to both classes and methods. When applied to a class, every public method of the class will be transactional. It manages transaction boundaries, start, commit, and rollback.

#### `Q` Describe what happens if an exception is thrown in a method annotated with `@Transactional`. How does Spring handle the transaction in this case??
`A` By default, Spring rolls back the transaction if an unchecked exception (e.g., a subclass of RuntimeException) is thrown. Checked exceptions do not trigger a rollback unless explicitly specified.

#### `Q` What is transaction propagation, and what are some of the propagation behaviors available in Spring? Provide examples of when you might use different propagation levels.?
`A` Transaction propagation in Spring defines how transactions relate to one another, particularly when a method is called within an existing transaction context. Here are the different propagation behaviors available in Spring:
- `PROPAGATION_REQUIRED`: Joins the existing transaction if one exists; otherwise, it starts a new transaction. This is the most common propagation and is used when you want to ensure that all operations occur within a single transaction context. For instance, updating multiple related records where atomicity is necessary.
- `PROPAGATION_REQUIRES_NEW`: Gets out of the current transaction (if any) and starts a new transaction. Useful when you need the called method to run in its independent transaction. For example, logging an audit trail or updating a logging table, which should always commit even if the main transaction fails.
- `PROPAGATION_SUPPORTS`: Joins the existing transaction if one exists; otherwise, it proceeds without a transaction. Ideal for read-only operations that can execute with or without a transaction. Specially data that doesn't change frequently.
- `PROPAGATION_NOT_SUPPORTED`: Suspends the existing transaction, if one exists, and executes the method non-transactional.Used when a certain operation must not run within a transaction context, such as performing time-consuming operations like data exporting.
- `PROPAGATION_NEVER`: Executes non-transactional and throws an exception if an existing transaction is present. Similar use cases like previous one.
- `PROPAGATION_MANDATORY`: Joins the existing transaction; throws an exception if no transaction is present. This is used when an operation absolutely requires an existing transaction, such as operations that must be part of a composite larger transaction initiated elsewhere.
- `PROPAGATION_NESTED`: Executes within a nested transaction if a current transaction exists; otherwise, it behaves like PROPAGATION_REQUIRED. Ideal for scenarios where you need rollback capabilities at a finer granularity than the main transaction scope allows. For example, processing a bulk operation where each item process should roll back independently if it fails, without affecting the overall bulk transaction.

#### `Q` How does Spring Boot handle transaction isolation levels? Explain the different isolation levels and their impact on database transactions.?
`A` Spring Boot allows setting transaction isolation levels through the `@Transactional` annotation. Common levels include `READ_UNCOMMITTED`, `READ_COMMITTED`, `REPEATABLE_READ`, and `SERIALIZABLE`, each providing different balances of consistency and concurrency.
- `READ_UNCOMMITTED`: Allows a transaction to read uncommitted changes made by other transactions, leading to "dirty reads." Rarely used in practice due to reliability issues but might be applicable for read-intensive applications where performance is more critical than data accuracy. Impact: High concurrency but with risks of reading inconsistent data.
- `READ_COMMITTED`: Only read data committed by other transactions, preventing dirty reads. Commonly used in production environments where a balance between data consistency and performance is needed. Impact: Prevents dirty reads but phantoms and non-repeatable reads can still occur.
- `REPEATABLE_READ`: Ensures that if a transaction reads a row, subsequent reads will return the same data, preventing non-repeatable reads but not phantoms. Suitable for applications needing consistent reads within a transaction, like generating reports or invoices.
- `SERIALIZABLE`: Transactions are completely isolated from each other, locking both read and write operations to prevent any concurrent access issues. Used in scenarios where absolute consistency is critical, such as financial applications where precise balances must be maintained.

#### `Q` Can you have a transaction that involves multiple databases in Spring Boot? How would you manage such a transaction??
`A` Yes, through XA transactions or distributed transaction managers, you can manage transactions across multiple data sources by using JTA (Java Transaction API).

#### `Q` What are some potential pitfalls or common mistakes when using transactions in Spring Boot applications??
`A` Common pitfalls include not handling exceptions correctly for transaction rollback,
misunderstanding transaction isolation level impacts, or all methods in a class not being
public, which can prevent @Transactional from working effectively.

Read more on @Transaction pitfalls:
- https://www.linkedin.com/pulse/understanding-springsvtransactional-tips-best-aleksei-gavrilichev-roclf/
- https://dzone.com/articles/spring-transactional-and-private-methods-snippet
- https://medium.com/@safa_ertekin/common-transaction-propagation-pitfalls-in-spring-framework-2378ee7d6521

Why these issues happens? Well, Spring AOP works by creating a proxy around the bean. By default, this proxy intercepts calls to public methods to apply the transactional behavior defined by @Transactional.
Only public methods are intercepted when using proxy-based AOP. Calls to private, protected, or package-private methods from outside the class will not be intercepted by the proxy, which means that @Transactional annotations on these methods won't have any effect.
If a transactional method makes an internal call to another method within the same class, and this method is not public (or not explicitly transactional when called externally), that call will not be intercepted by the proxy. As a result, the transaction management behavior, such as starting a new transaction, joining an existing one, or rolling back, will not be applied.
For transactions to work, methods need to be public and accessed externally — typically through another transactional method or service layer.

#### `Q` What if DB crash during a commit?
`A` Write ahead log and recovery procedures are applied to complete or rollback the commit.

#### `Q` Why read only transactions are used?
`A` A read-only transaction ensures a consistent snapshot of the data without allowing modifications, optimizing for performance and isolation, while a regular read might not guarantee the same isolation or consistency.
A read-only transaction ensures consistency by locking in a stable snapshot of the database at the start of the transaction, regardless of ongoing updates, thus preventing it from seeing partial changes from other operations.

#### `Q` How costly is read only lock/snapshot?
`A` For read-only transactions, the "lock" is generally not a traditional lock that blocks writes; instead, it's often more of a logical isolation mechanism like a snapshot that doesn't interfere significantly with write operations.
Therefore, the cost is usually minimal in terms of locking overhead, primarily impacting resource usage for maintaining the snapshot.
In systems that use Multi version Concurrency Control (MVCC), this approach allows high read concurrency with minimal locking impact.

#### `Q` Mention some pros and cons of MVCC?
`A` `Pros:` Multi version concurrency control allows version of data ensuring readers without locking or blocking another write or read, it also reduces dead locks.
`Cons:` Increased storage needs, complex garbage collection, more space for storing & more CPU for complex write, garbage collection and read algorithms.
Note that, all common SQL databases use MVCC. Some NoSQL databases like MongoDB and Cassandra have their tailored concurrency control methods, which might share characteristics with MVCC but aren't strictly MVCC.

#### `Q` Question: Explain the difference between @Transactional and using TransactionTemplate?
`A` @Transactional supports declarative transaction management through annotations, allowing transaction management to be separated from business code. TransactionTemplate provides programmatic transaction management, where transactions are manually controlled in code, offering more precise transaction boundary control in complex scenarios.

#### `Q` What is the default isolation level in Spring if not specified??
`A` If not specified, Spring uses the default isolation level of the underlying database, which is typically READ_COMMITTED.










## `Indexing` 
#### `Q` What are indexes and their different types?
`A` Indexes are data structures that improve the speed of data retrieval operations within a database table at the cost of additional writes and storage space to maintain the index data structure.
- Primary Index: Index using the primary key.
- Unique Index: This index does not allow the field to have duplicate values if the column is unique indexed.
- Secondary Index / Non-unique index: Unlike the primary index, a secondary index may not be based on a unique key, and multiple entries can have the same key. It is used to access data in sequential order, but using non-primary keys.
- Composite Index: Also known as a concatenated index, involves multiple columns. Composite indexes can be particularly useful for queries that test all the columns in the index, or prefix the columns.
- Clustered Index: In a clustered index, the order of the physical data records on the disk is the same as the index order. There can be only one clustered index per table because data rows themselves can be stored in only one order. The primary key of a table is a great candidate for a clustered index.
- Bitmap Index: Bitmap indexes are typically used for columns with low cardinality, which means the column has a very few unique values. They use bitmaps (arrays of binary digits) to represent the existence of data values, making it highly efficient for indexing columns with Boolean values or null status.
- Partial Index: Also known as a filtered index, it only indexes a subset of the data in a table.
- Full-text Index: Used on columns holding large strings or documents, full-text indexes allow for the searching of words or phrases within the strings. This type of indexing is common in applications handling large textual data like logs, documents, and descriptions allowing complex search queries.
- Covering Index: A covering index includes all the columns needed for a query to be processed. Essentially, it "covers" the query.

#### `Q` What is the difference between clustered and non-clustered indexes?
`A` A clustered index determines the physical order of data in a table and stores it in the index leaf nodes. There is only one clustered index per table, which can improve the performance of data access by reducing disk I/O. A non-clustered index, however, creates a separate structure to hold the index data, which includes pointers to the location of the actual data. This allows for multiple non-clustered indexes per table but can result in slower data access compared to clustered indexes.

#### `Q` Explain the concept of index selectivity and its importance.
`A` Index selectivity  is defined as the ratio of distinct values to the total number of values in a column. High selectivity (a higher ratio) means the index can more effectively optimize queries, as the index entries are more unique

#### `Q` Can you create an index on a view?
`A` Generally, you can't create an index on a view because views are virtual tables and don't store any data. However, some databases allow indexing on materialized views, which contain the actual data. This is useful when the view is used frequently, and you want to improve its performance.

#### `Q` Explain "Index Covering" for queries.
`A` Index covering occurs when all the columns used in a query are part of one index. Essentially, if the index contains all required fields for a query output, the database system can retrieve data from the index itself without needing to look up the full records in the table, resulting in much faster query execution.

#### `Q` What is an inverted index?
`A` An inverted index is a database index that stores a mapping from content to its location in a database file, table, or a document. It's commonly used in document stores and full-text search engines to allow fast text searches.

#### `Q` What are partial indexes and their use cases?
`A` Partial indexes are indexes built on a subset of a database table's rows, determined by a specified filter. They are useful when only a small subsection of the data needs to be frequently accessed or queried, thereby improving efficiency and reducing index size.

#### `Q` How do functional-based indexes work, and when are they used?
`A` Functional-based indexes are used to index the results of a function applied to one or more columns. They are useful when queries commonly involve conditions on transformations of data (like uppercase strings).

#### `Q` What is index cardinality, and why is it important?
`A` Index cardinality refers to the uniqueness of data values represented in an index. High cardinality indexes (with many unique values) are typically more efficient at data retrieval than low cardinality indexes (with many repeated values).

#### `Q` What is an index hint, and why might one use it?
`A` An index hint is a directive used in SQL queries to suggest to the query optimizer to use a specific index when executing the query. It can be used to improve performance, particularly in complex queries where the optimizer might not choose the most efficient execution plan.

#### `Q`  What is a filtered index?
`A` A filtered index is an index with a WHERE clause, essentially indexing a subset of rows in a table. This can be more space-efficient and perform better than full-table indexes for queries on sizable but specific segments of data.

#### `Q` What challenges might arise with indexes when scaling horizontally (sharding)?
`A` When sharding a database, indexes must be maintained on each shard, which can lead to inconsistencies if not carefully managed. Cross-shard queries might become slower and more complex, as they may require aggregating index results from multiple shards.

#### `Q` Explain how virtual indexes are used in database testing.
`A` Virtual indexes are hypothetical indexes that do not actually exist in the database but are considered by the query optimizer during the planning phase to predict whether creating an actual index might help query performance. They are used in testing to avoid the overhead of index creation for performance tuning purposes only.

#### `Q` How do you decide which columns to index?
`A` Columns that are frequently used in the WHERE clause, JOIN conditions, or as part of an ORDER BY or GROUP BY clause are good candidates for indexing.

#### `Q` What is an index seek?
`A` An index seek occurs when the query optimizer uses the index to find specific index entries quickly; usually, this happens when a query's WHERE clause matches columns that are part of a useful index.

#### `Q` What is partitioned indexing?
`A` Partitioned indexing involves partitioning an index in a similar way as partitioning a table. This can improve the performance and manageability of indexes by restricting index maintenance and search to relevant partitions.

#### `Q` What is index fragmentation?
`A` Index fragmentation refers to the physical fragmentation of an index on disk or in memory. In this case, the index’s data pages may not be sequential or unnecessary gaps may occur. This can slow down access to indexes by database management systems (DBMS) and negatively impact query performance.

#### `Q` What is an adaptive hash index?
`A` An adaptive hash index is a feature in some databases that automatically creates hash indexes on certain table partitions based on observed query patterns, improving performance with no prior configuration required.

#### `Q` How do you perform index tuning?
`A` Index tuning involves analyzing query performance, identifying bottlenecks, and then creating, modifying, or dropping indexes based on the actual usage pattern. Tools and scripts that analyze query plans and index usage can assist greatly in this process.

#### `Q` What are the types of index structures commonly used in databases?
`A` Common types of index structures include B-Trees, R-Trees, Hash tables, and Bitmaps. Each serves different use cases, from general-purpose indexing and range queries to spatial data searching and indexing low-cardinality columns.

#### `Q` What is the difference between a dense index and a sparse index?
`A` A dense index has an index entry for every single record in the data file, whereas a sparse index only contains entries for some records. Sparse indexes require less space and are suitable for cases where records are sequentially stored.









## `Partitioning`
#### `Q` What are the types of partitioning available in SQL databases?
`A` Common types include range partitioning, list partitioning, hash partitioning, and composite partitioning.

#### `Q` What are global and local indexes in partitioned tables?
`A` Global indexes span all partitions of a table, while local indexes are specific to individual partitions.

#### `Q` Is partitioning helpful for write-heavy applications?
`A` Yes, by reducing lock contention and distributing the load across partitions, it can enhance performance in write-heavy applications.

#### `Q` Can you convert a non-partitioned table to a partitioned table?
`A` Yes, most databases allow altering of table structures to introduce partitioning, though specific steps can vary.

#### `Q` Explain the concept of partition pruning.
`A` Partition pruning is a performance optimization technique where a database query optimizer automatically skips scanning partitions that do not match the query criteria.

#### `Q` How does hash partitioning work?
`A` Hash partitioning distributes data across partitions based on a hash function applied to the partition key, aiming for a uniform distribution of data.

#### `Q` What is composite partitioning?
`A` Composite partitioning involves combining two or more partitioning strategies, such as first applying range partitioning and then hash partitioning within those partitions.

#### `Q` If a table is partitioned in 10 sub tables and you run a query which doesn't involve partition key, then how the query will be executed and data will be fetched.
`A` If a query is executed on a partitioned table without involving the partition key, the database system will need to perform a full table scan across all partitions. This is because the absence of the partition key in the query conditions means that the system cannot determine which partition(s) contain the relevant data. Consequently, the database engine must search through each subtable (partition) to retrieve the necessary data. This type of operation is less efficient than queries that use the partition key.










## `Sharding`
#### `Q` What are the types of sharding?
`A` Key types include horizontal sharding (splitting rows), vertical sharding (splitting columns), and functional sharding (dividing data by business function).

#### `Q` Explain consistent hashing used in sharding.
`A` Consistent hashing is a technique used to distribute entries among a fixed number of buckets (shards) with minimal movement of entries when new buckets are added or removed.

#### `Q` How do you handle transactions in a sharded environment?
`A` Handling transactions across shards can be complex and typically involves mechanisms to ensure atomicity and consistency such as two-phase commits or compensating transactions or saga patterns.

#### `Q`  Can you reshard data once sharding is implemented?
`A` Yes, data can be resharded but it generally involves significant complexity and overhead. It may require redistribution of data and application downtime.

#### `Q` How do you query data in a sharded environment?
`A` Queries may need to be executed across multiple shards. Applications often use a shard map to route queries to the appropriate shard or aggregate results from multiple shards.

#### `Q` When should vertical sharding be considered over horizontal sharding?
`A` Vertical sharding can be considered when different tables of a database have different usage patterns or when certain columns are accessed much more frequently than others.

#### `Q` How do ORMs handle sharded databases?
`A`  ORMs can handle sharded databases through shard-aware plugins or extensions that can route data operations to the correct shard based on shard keys.

#### `Q` How does sharding interact with cloud services?
`A` Many cloud databases support automatic sharding, allowing applications to scale horizontally across multiple instances seamlessly as demand increases.












## `Replication`
#### `Q` What are the main types of replication?
`A` The main types include master-slave replication, peer-to-peer or multi master replication, and snapshot replication.

#### `Q` What are the benefits of database replication?
`A` Benefits include increased data availability, fault tolerance, load balancing, and separate environments for critical operations like reporting and backup without impacting production.

#### `Q` What challenges can occur with database replication?
`A` Challenges include managing data consistency, conflict resolution, replication lag, and the overhead of maintaining multiple synchronized systems.

#### `Q` What is replication lag and how can it be mitigated?
`A` Replication lag is the delay between an operation being performed on the primary server and being replicated to the secondary server(s). It can be mitigated by optimizing the network, increasing system resources, or simplifying transactions.

#### `Q` Explain synchronous versus asynchronous replication.
`A` Synchronous replication ensures that transactions must be committed on all replicas at the same time. Asynchronous replication allows transactions to be replicated in the background, potentially leading to replication lag but less impact on performance.

#### `Q` How do you handle conflict resolution in bidirectional replication?
`A` Conflict resolution strategies might include "last-write wins," timestamp-based resolution, or merging changes according to application-specific rules to resolve conflicts in bidirectional replication.

#### `Q` Explain log shipping as a form of replication.
`A` Log shipping involves automatic sending of transaction log files from one server to another. The secondary server restores the logs, keeping the database as an exact copy of the primary.

#### `Q` What is physical replication in database systems, and how does it work?
`A` Physical replication involves copying and synchronizing the exact binary data from one database to another. This type of replication is usually implemented at the storage or file-system level and captures all changes made in the database, including data, schema, and index modifications. It ensures a byte-for-byte copy between the master and the replica, typically using a streaming replication method that can be synchronous or asynchronous.

#### `Q` How does physical replication handle failover scenarios?
`A`  Physical replication is often used in high-availability setups to handle failover scenarios. If the primary database fails, one of the physically replicated secondary databases can take over as the primary. This switchover can be automated or manually triggered, depending on the system configuration, to maintain service availability without significant downtime.

#### `Q`  Compare the performance implications of physical and logical replication.
`A` Physical replication generally has less overhead on the source system since it directly copies binary data, often leveraging existing database mechanisms and infrastructure. This makes it very efficient in terms of performance. However, because it replicates all data, it might demand more bandwidth and storage. Logical replication, on the other hand, may introduce additional CPU overhead on the source database as it needs to transform transaction logs into a logical format and can apply filters or transformations. This additional processing can impact database performance but provides more control over what is replicated, leading to potentially lower network and storage requirements if only a subset of the data needs to be replicated.







## `Concurrency`

#### `Q` What is database concurrency?
`A` Database concurrency refers to the ability of a database system to allow multiple transactions to access the same data simultaneously without compromising data integrity.

#### `Q` What are the common issues faced with database concurrency?
`A` Common issues include dirty reads, non-repeatable reads, phantom reads, and lost updates.

#### `Q` How does optimistic concurrency control work?
`A` Optimistic concurrency control assumes that multiple transactions can complete without interfering with each other, checking for conflicts only at commit time.

#### `Q` How does pessimistic concurrency control work?
`A` Pessimistic concurrency control locks resources during a transaction to prevent other transactions from accessing the same resources, thus preventing conflicts.

#### `Q` What is a lock in database concurrency?
`A` A lock is a mechanism to control access to data in a database to prevent concurrent transactions from interfering with each other.

#### `Q` What is a deadlock in a database?
`A` A deadlock occurs when two or more transactions are waiting indefinitely for one another to release locks.

#### `Q` How can deadlocks be prevented?
`A` Deadlocks can be prevented by acquiring locks in a consistent order, using timeouts, or implementing deadlock detection and resolution strategies.

#### `Q` What is a two-phase locking protocol?
`A` Two-phase locking protocol ensures serializability by dividing the transaction process into two phases: lock acquisition and lock release.

#### `Q` What is timestamp-based concurrency control?
`A` Timestamp-based concurrency control assigns a unique timestamp to each transaction and uses it to order transactions to ensure consistency.

#### `Q` What is Multi-Version Concurrency Control (MVCC)?
`A` MVCC allows multiple versions of data to exist simultaneously, enabling readers to access data without blocking writers, thus enhancing concurrency.

#### `Q` How do isolation levels relate to concurrency?
`A` Isolation levels determine how transaction integrity is visible to other transactions and can affect the concurrency and performance of a system.

#### `Q` What is a lock escalation?
`A` Lock escalation is a process of converting multiple fine-grained locks into a single coarse-grained lock to reduce overhead.

#### `Q` What is a write skew?
`A` Write skew occurs when two transactions read overlapping data and update non-overlapping data, violating consistency.

#### `Q` What is lock contention?
`A` Lock contention happens when multiple transactions compete for the same lock, leading to delays and reduced concurrency.

#### `Q` What is a shared lock?
`A` A shared lock allows multiple transactions to read a resource but not modify it.

#### `Q` What is an exclusive lock?
`A` An exclusive lock prevents other transactions from accessing the locked resource for both reading and writing.

#### `Q` How do databases handle high concurrency scenarios?
`A` Databases handle high concurrency through techniques like indexing, partitioning, replication, and tuning isolation levels to balance performance and consistency.

#### `Q` How does a database system handle long-running transactions that hold locks?
`A` The database system will either abort the transaction or use a timeout to release the locks.

#### `Q` What is the difference between a shared lock and an intent lock?
`A` A shared lock allows multiple transactions to read a resource but not modify it, while an intent lock is used to prevent other transactions from acquiring a shared lock on the same resource.

#### `Q` What is a lock timeout?
`A` A lock timeout is a mechanism used to automatically release a lock after a specified time has elapsed.

#### `Q` What is a deadlock detector?
`A` A deadlock detector is a mechanism that monitors the database for potential deadlocks and can take actions to prevent or resolve them.

#### `Q` What is the difference between optimistic and pessimistic locking?
`A` Optimistic locking assumes that multiple transactions can complete without interfering with each other, while pessimistic locking assumes that transactions will conflict and locks resources accordingly.

#### `Q` What is a lost update?
`A` A lost update occurs when two transactions update the same data simultaneously, and one of the transactions is lost or overwritten.

#### `Q` How do you prevent lost updates?
`A` Lost updates can be prevented using optimistic or pessimistic locking, or by using a timestamp or version number to track changes.










## `Security`
#### `Q` How do you develop and enforce database confidentiality policies?
`A` By assessing data sensitivity, defining access controls, and implementing layers of security such as encryption and user authentication. Enforcing these policies involves regular audits, updating security protocols as per compliance standards, and training users on data privacy practices.

#### `Q` What is SQL injection and how can it be prevented?
`A` SQL injection is a code injection technique that exploits vulnerabilities in an application's software by inserting malicious SQL code into query fields. It can be prevented by using parameterized queries or prepared statements, validating user inputs, and employing web application firewalls.

#### `Q` What are some best practices for database encryption?
`A` Best practices include encrypting sensitive data both in transit and at rest, using strong encryption algorithms, managing encryption keys securely, and regularly updating encryption protocols.

#### `Q` How do you secure user authentication in a database system?
`A` Implement strong password policies, use multi-factor authentication, store passwords with cryptographic hashing, and regularly audit authentication logs for suspicious activities.

#### `Q` What is the principle of least privilege and how is it applied in databases?
`A` The principle of least privilege involves granting users the minimum level of access necessary for their roles. In databases, this means setting up roles and permissions carefully and regularly reviewing them to minimize security risks.

#### `Q` How do you protect against unauthorized database access?
`A` Use strong authentication methods, enforce strict access controls, regularly update and patch database systems, and monitor access logs for unusual activities.

#### `Q` What is a database firewall and how does it enhance security?
`A` A database firewall is a security barrier that monitors and controls database traffic based on predefined security rules. It helps prevent unauthorized access, SQL injection, and other attacks by filtering suspicious queries.

#### `Q` How can data masking be used to enhance database security?
`A` Data masking obscures sensitive data by replacing it with similar, non-sensitive data. It's used in non-production environments to protect real data from exposure during development and testing.

#### `Q` What role does auditing play in database security?
`A` Auditing involves tracking and recording database activities to detect and respond to suspicious actions. It helps ensure compliance, identify security breaches, and provide forensic evidence.

#### `Q` How do you ensure secure database backups?
`A` Secure backups by encrypting them, storing them in a separate, secure location, restricting access to backup files, and regularly testing backup restoration processes.

#### `Q` What are common strategies for database patch management?
`A` Regularly update database systems with the latest security patches, test patches in a staging environment before deployment, and implement automated patch management tools.

#### `Q` How does role-based access control enhance database security?
`A` Role-based access control assigns permissions to roles rather than individual users, simplifying management and ensuring that users have appropriate access based on their job functions.

#### `Q` What is database hardening, and how is it performed?
`A` Database hardening involves securing a database system by reducing its attack surface. This includes disabling unnecessary services, applying patches, configuring security settings, and regularly auditing the system.

#### `Q` How do you secure database connections?
`A` Use SSL/TLS for encrypted connections, restrict database access to specific IP addresses, and use secure connection strings with minimal permissions.

#### `Q` What is the importance of logging in database security?
`A` Logging provides a record of database activities, helping to detect unauthorized access, monitor user actions, and provide data for forensic analysis after a security incident.

#### `Q` How do you implement data loss prevention in databases?
`A` Use encryption, regular backups, access controls, and monitoring tools to prevent unauthorized access and accidental data loss.

#### `Q` What measures can be taken to secure a database server?
`A` Measures include physical security controls, network segmentation, patch management, and regular security audits.

#### `Q` How can you protect against insider threats in database security?
`A` Implement strict access controls, monitor user activities, use data masking, and conduct regular security training for employees.

#### `Q` What is the role of a database security policy?
`A` A database security policy defines the rules and procedures for protecting database systems and data, ensuring compliance with legal and organizational standards.

#### `Q` How do you handle database security in a cloud environment?
`A` Use cloud provider security features, encrypt data, configure access controls, and monitor cloud database activities for suspicious behavior.

#### `Q` What are the benefits of using a Virtual Private Network (VPN) for database access?
`A` A VPN encrypts data traffic, providing a secure connection to the database and protecting it from eavesdropping and unauthorized access.











## `Ad-Hoc`
#### `Q` How do you know when to follow "procedural" logic?
`A` Procedural logic is suited to tasks that involve sequential steps, such as data processing routines, complex calculations, and detailed instructions that must be executed in a specific order. It’s particularly useful when exact procedures must be followed to achieve the desired outcomes, ensuring precision and control over program flow.

#### `Q`Should you use create table if not exist (or ORM based table creation) from the web server?
`A` If I do this, the web server will have full privilege which is bad because the web server has drop
permission so SQL injection can destroy everything, another problem is what if there is another
web server that use the same table? There will be imbalance of access. So what I'll do is create
database independently then make connection pool with specific permissions for the web server.
We can even create multiple pools for each web server even for each of the table but that may
add extra level of configurations. Need to find a middle point in-between based on business.

#### `Q` What is the best way of connecting database from server?
`A` Use SSL, use very long and unpredictable DB password, store the configuration info in some
vault area which will be utilized by the web server.

#### `Q` What is Homomorphic encryption?
`A` We can not just save everything encrypted into database prioritizing security because while
querying, we'll need comparison and filter logic to be applied on plain text, can't do those on
cypher. Homomorphic encryption, introduced by IBM, so-called future of security, actually can
apply those on a cypher text! Of course, it is not magic, extra level of processing is needed, so
it is ridiculously slow yet. That's why it is called ultimate security tech for 'future'.

### `Q` What is de-normalization in database design?
`A` De-normalization involves adding redundant data or grouping data to a database to speed up read operations, often at the expense of additional storage and more complex data updates. It's typically used to reduce the number of table joins required in queries.

#### `Q` In composite indexing what orders are best?
`A` When we're indexing, we don't care about space. We should consider how the B tree may
respond quick, so we should put more selective(unique, less-repeat) field on the left. Let's assume
an index of productId and category, if we put category on left then there will be only a few first
level nodes each one containing huge amount of second. But using productId will do the reverse
and result faster.

#### `Q` How UUID is bad?
`A` UUIDs can be problematic primarily due to their size (16 bytes) compared to traditional integer IDs (typically 4 bytes),
which can lead to increased storage and index size. This can impact performance, particularly in database operations
involving joins (string comparing is costly - O(N)), indexing (no order, B-tree performs like hashing as there is no chance of ranged query),
and data retrieval. Additionally, UUIDs are not naturally ordered, which can lead to inefficiencies in data storage and fragmentation.

#### `Q` How indexes that are too large for RAM stored in disc and accessed?
`A` Indexes too large to fit into RAM are stored on disk. When a query is executed, the database retrieves the needed
parts of the index from disk into RAM. This process can slow down data retrieval time compared to when everything fits into RAM.
Databases use mechanisms like caching and buffer management to optimize the performance of disk-based index access.

#### `Q` What is better, making a column non-null or putting null values or putting dummy value like 0?
`A` Generally, making a column non-null is better if you expect a value for every record; but if a field is avoidably
nullable then a nullable column is better choice than putting dummy values like '0'. Because even when put an integer 0,
it takes 8 bits memory but while storing null in a field databases use very efficient low memory consuming techniques like bit map.
I.e.: A record with 8 null fields will take equal space to storing an integer by using binary bit map.

#### `Q` How does postgres determine a record in a row is null?
`A` Postgres uses a special area in the row's header known as the null bitmap, where each nullable column has a bit that indicates
whether the column is null or not. If the bit is set, Postgres interprets that column as having a null value.

#### `Q` Is there can be any difference in select count(*), select count(col_name) in a single column table? If yes then why would!? And when?
`A` Yes, there can be a difference. SELECT COUNT(*) counts all rows in the table, including those with null values. In contrast,
SELECT COUNT(col_name) counts only the rows where col_name is not null. The difference will be apparent in cases where col_name contains null values.

#### `Q` Why select * from t where c is null gives result but select * from t where c in [null] doesn't?
`A` This occurs because SQL's IN clause does not process NULL in the expected manner. NULL represents an unknown value,
and IN [NULL] does not match anything, not even other NULLs. Instead, IS NULL must specifically be used to check for null values.

#### `Q` Do databases support nulls in the index?
`A` Most databases, including PostgreSQL and MySQL, do allow null values to be indexed. However, the way nulls are handled in indexes
can vary between database systems. Some databases can index null values by default, while others require specific configuration or handling.
