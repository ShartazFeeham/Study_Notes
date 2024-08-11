## Transaction
-	Multiple queries treated as a single unit of work.
-	Transaction lifecycle consists of 4 parts internally: BEGIN, COMMIT (Store in memory/RAM), WRITE (Store in disk).
-	However, these can differ by DB engines, each DB engine has its own customized and complex transaction management system. Some focus more on commits some focus on crashes while some try to balance.
-	For COMMIT & WRITE there can be two approaches basically.
-	Each smaller unit is committed and written in disk -> needs lot of I/O, very fast, hard to rollback, used by Postgres
-	Every commit is prepared first then all are stored in disk -> slower, less IO, low failure chance, easier rollback
-	But, what if the DB crash during a commit?
-	Read only transaction is also normal, because it gives snapshot of a moment, without transaction, in regular reads, things may get changed in the middle.

**Example of a transaction**

![img_1.png](img_1.png)