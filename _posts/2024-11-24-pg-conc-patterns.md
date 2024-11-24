---
layout: post
title: "Database Concurrency Patterns For Developers"
categories: misc
---

I have been developing applications interacting with databases for many years now and I have always taken concurrency for granted. I looked at the database as a black box and I never had to think about concurrency issues assuming that the database would handle it for me.

My interactions with the DB have mostly been limited to basic CRUD operations—such as inserting and retrieving data, performing stateless updates and deleting records, typically for individual records. These usecases has always worked well for me, so I never gave it much thought.

Until I had to work on a problem to implement a complex business logic that required a careful
handling of concurrency on top of the database and I could not reason about the correctness of my approache. 
I was just hopping from a Stack Overflow post to another, not sure if I understood the problem and only trying to find a way 
to take locks as I was mentally drawing a parallel between concurrency at the database level and general tools and patterns for concurrent programming.

So I took the time and effort to understand different guarantees and tools provided by the database and how they work and it helped me a lot to reason about the correctness of my queries and allowed me to better understand the trade-offs of different approaches.

In this post, I will share some what I have learned and encourage you to dive deeper into the topic if needed.

⚠️ All information below is valid for PostgreSQL and might not be applicable to other databases.

#### <u>[Teaser questions to get you started]</u>

If you have doubts or difficulties answering them, then this blog post if for you:

##### <span style="color:red"><u>Example 1:</u></span>

Imagine you have two concurrent queries that both want to update some record as the following:

```sql
UPDATE records
SET name = 'aaa', type = 'a'
WHERE id = 1;
```

```sql
UPDATE records
SET name = 'bbb', type = 'b'
WHERE id = 1;
```

Is it possible to have a record with name `aaa` and type `b` or with a name `bbb` and type `a` at the end of both queries like this?

```
+----+------+------+
| id | name | type |
+----+------+------+
|  1 | aaa  |  b   |
+----+------+------+
```

##### <span style="color:red"><u>Example 2:</u></span>

Now imagine you have a counter that is incremented to account for the numbers of emails sent to a given user. 

Would it work if you have many concurrent queries like the following trying to increment the counter at the same time?

```sql
UPDATE users
SET emails_sent = emails_sent + 1
WHERE id = 1;
```

what's is the difference with the following query?

```sql
BEGIN; -- start a transaction

UPDATE users
SET emails_sent = emails_sent + 1
WHERE id = 1;

END;
```

And this one?

```sql
BEGIN;
-- you can't directly assign variables in PG this way but assume this is done in code
current_emails_sent := (SELECT emails_sent FROM users WHERE id = 1);

UPDATE users
SET emails_sent = current_emails_sent + 1
WHERE id = 1;

END;
```

Which one is the correct one and why?

##### <span style="color:red"><u>Example 3:</u></span>

Here is another example: let's say you want to get students in the computer science department and you have a table with the following data:

```
+------------+------------+-------------------+
| ID         | Name       | Department        |
+------------+------------+-------------------+
| 1001       | Alice      | Computer Science  |
| 1002       | Charlie    | Mathematics       |
| 1003       | Bob        | Computer Science  |
+------------+------------+-------------------+
```

You run the following query:

```sql
SELECT * FROM students WHERE department = 'Computer Science';
```

Imagine another concurrent query is running and updates the department of `Bob` to `Mathematics` before the first query gets to the last row. What would happen? 
Would `Bob` be in the result set or not? 
In other words, what would happend if the second query makes the update before the first query gets to the last row?

##### <span style="color:red"><u>Example 4:</u></span>

Here is a more subtle example: let's say your API receives queries to increment the number of games of two players. 
You receive two API queries with the same players (1 ans 2). 

You open two transactions and increment the number of games for each player by running following identical queries in parallel:

```sql
BEGIN;

UPDATE games SET nb_games = nb_games + 1 WHERE player_id IN (1, 2);

END;
```

and this one:

```sql
BEGIN;

UPDATE games SET nb_games = nb_games + 1 WHERE player_id IN (1, 2);

END;
```

What could happen?

#### <u>[Why is concurrency control important at the database level?]</u>

It's all about <u>consistency</u>. What is consistency? It means the data is in a valid state. What does it mean for the data to be valid? 
Well it depends on your application and the business logic you are implementing.


Some consistency guarantees can (and should) be expressed in the form of constraints at the database level. 
For example, you can enforce that a column can be used as an identifier for individual records by being unique and not null by telling the database that it is a primary key. 
It's the job of the database to enforce this constraint and not allow you to violate it and you can be sure that the data is always valid from this point of view.

Not all consistency guarantees can be easily or even possibly expressed at the database level in the form of constraints. 
So the burden is on the application (and thus you) to ensure that the data is always valid. 
For example, booking a room should only be possible two weeks in advance except for clients with a premium subscription. 
You have to make sure this is enforced when receiving a booking request at the application level.


A DB is supposed to run operations. If all operations take the data from a valid state to a another valid state, 
running them serially (one by one) will ensure that the data is always in a valid state as shown in the following diagram:

![consistency diagram](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_transistion.png)

Sometimes, some business logic maight requires mutliple operations to be implemented.
This  can need to break consistency momentarily. 

For example, you debit 10$ from one account (first operation) and credit it to another (second operation). 
You can see that at the moment <u>after the debit</u> and <u>before crediting the second account</u> the data not in a valid state (10$ just disappeared).


What happens if there is a failure in the second operation? Well, consistency is broken and the gate to hell is wide open.

![consistency diagram bad state](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_transistion_bad_state.png)

#### We need a transaction!

To avoid the previous situation, the database provides a way to group operations together and make sure they are always ALL executed or NONE of them. 
If none of them are executed we will stay at the same valid state as before (assuming individual transactions preserve data validity), 
if they all are executed, we will end up in a valid state. This is called a **transaction**. 

Transactions are then a group of operations that are executed as a single unit of work (ATOMIC) and can only take the data from a CONSISTENT state to a another CONSISTENT state. These are your first two letters of <u>AC</u>ID.

![consistency diagram transaction](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_transistion_tx.png)

#### Everything is a transaction!

PostgreSQL treats each statement as a transaction. If you issue a statement, it will be wrapped in a transaction and committed if successful or rolled back if an error occurs. If we go back to our teaser questions, these two are actually equivalent:

```sql
UPDATE users
SET emails_sent = emails_sent + 1
WHERE id = 1;
```

```sql
BEGIN;

UPDATE users
SET emails_sent = emails_sent + 1
WHERE id = 1;

END;
```

We can also be sure then that the following query will be atomic, meaning that either both fields will be updated or none of them will in case of a failure:

```sql
UPDATE records
SET name = 'aaa', type = 'a'
WHERE id = 1;
```

#### Concurrency is desirable but can be dangerous!

The database needs to handle high throughput of transactions. This means that it has to execute transactions concurrently. 
What's the problem with that? A set of transactions that will leave the DB in a valid state if executed <u>serially</u> 
can leave it in an invalid state if executed <u>concurrently</u>. 

Take the following example: A user has **100$** in their account. 

Two concurrent transactions try to debit **10$** from the account at the same time and credit it to another account. 

They both see **100$** and succeed but they both try to update the balance of the user to *90$* (100$ - 10$) at the end. 

This is an inconsistent state because **10$** was created from thin air and the user should actually have had **80$** left in their account.

What happenns we we run both transactions serially? first one and then the second one (or vice versa), the balance would be consistent. 

This leads us to th following conclusion:
> A valid concurrent execution of a set of operations is valid if and only if the final state 
> corresponds to the outcome of one of the possible serial executions of the operations (because we know that a serial execution of transactions leave the data in a valid state).

Here is another example: Imagine you have two transactions, one want to add **10$** to the balance and 
another one wants to increase it by **10%** as follows:

```sql
UPDATE users
SET balance = balance + 10
WHERE id = 1;
```

```sql
UPDATE users
SET balance = balance + balance * 0.1
WHERE id = 1;
```

If we run these two transactions concurrently, the execution is valid if and only if the balance at the end is either:

* **121$** (if the first transaction was executed first and then the second one)
* **120$** (if the second transaction was executed first and then the first one)

All other outcomes are invalid. **110$** is not valid for example (can happend if both read 100$ at the same time and then update). 

Invalid states are called <u>anomalies</u>.

#### <u>[Isolation!]</u>

We come to our third letter of AC<u>I</u>D. 
Isolation is the ability of the database to execute transactions concurrently without violating consistency, i.e. avoiding anomalies.

Here are some example anomalies:

##### <span style="color:red"><u>Lost update anomaly:</u></span>

Going back to our example where two transactions tried to debit **10$** from the user's account. 
One transaction updated the final balance to **90$** but the other one was not aware of that and continued thinking that the balance was still **100$**, so it updated it to **90$** too.

The update from the first transaction was lost.

![consistency diagram lost update](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_lost_update.png)


##### <span style="color:red"><u>Dirty read anomaly:</u></span>

Dirty read anomalies happen when a transaction reads data written by another concurrent transaction that was not committed.

Imagine a transaction wants to add **10$** to two accounts, it credits the first account but fails to credit the second one and thus it rollbacks.
A second concurrent transaction checks the balance of the first account, it sees 10$ and allows them to buy something before the first transaction rollbacks.


This is an anomaly because the first account never really had **10$** in it at the time of the second transaction.


![consistency diagram dirty read](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_dirty_read.png)

You should know that this anomaly is **not permitted in PostgreSQL**, and you don't have to worry about it at all.

In PostgresSQL the effects of a transaction can not be partially observed by another transaction while it's not yet committed. This is generally a consequence of <u>atomicity</u>.
From the point of view of other concurrent transactions, a transactions happens as a whole <u>instantaneously</u> (if not rolled back).
This means that when you write a transaction, you can be sure that others won't observe changes only after the transaction is committed.

##### <span style="color:red"><u>Non-repeatable read anomaly:</u></span>

Non-repeatable read anomalies happen when a transaction reads data, this data gets updated by another concurrent transaction that committed since the initial read
and then the first transaction reads the data again and finds that it has been modified.

Imagine you are constructing a dashboard from your data. You start a transaction that performs two select queries:

```sql
BEGIN;

-- compute the sum of salaries per department
SELECT department, SUM(salary) FROM employees GROUP BY department;
-- compute the sum of salaries for all employees for all departments
SELECT SUM(salary) FROM employee;

END;
```

Ideally, the result of the second query should be the same as the sum of salaries per department.

Like the following:

```
+------------+------------+
| department | salary     |
+------------+------------+
| Computer   | 1000       |
| Math       | 2000       |
+------------+------------+

+------------+
| salary     |
+------------+
| 3000       |
+------------+
```

Now imagine that between the two operations, another concurrent transaction adds other employees on the Computer department and commits it.

The results would be:

```
+------------+------------+
| department | salary     |
+------------+------------+
| Computer   | 1000       |
| Math       | 2000       |
+------------+------------+

+------------+
| salary     |
+------------+
| 3200       |
+------------+
```

This invalid state would happen if the second transactions added an employee in the Computer department with a salary of **200$**.

![consistency diagram non repeatable read](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_non_repeatable_read.png)


#### Isolation levels.

Anomalies mentioned above are not the only anomalies that can happen. There are many more that we know of and many that we don't know about.

The goal of the DB is to minimize possible anomalies by isolating transactions while keeping a good throughput of concurrent transactions.

Each DB has its way and tools of handling anomalies and isolation. 

Achieving full isolation on the database side while maintaining high throughput is hard and it can become impractical. 
So some of the responsibility falls on the application side, such as retrying aborted transactions due to serialization errors (more on this later).

In practice, databases implemented weaker isolation levels that are more forgiving to anomalies but also allow for more concurrency throughput.

The SQL standard defines four isolation levels:
1. **<span style="color:blue"><u>Read Uncommitted</u></span>**: This level is not supported by PostgresSQL and is not recommended. Dirty reads happend here.

2. **<span style="color:blue"><u>Read Committed</u></span>**: Does not allow for dirty reads but allows for non-repeatable reads, an operation in a transaction can read data modified by another transaction that has already committed before the current operation is run.

3. **<span style="color:blue"><u>Repeatable Read</u></span>**: A transaction only sees data committed before it's started to run. All operations in a transaction see a single snapshot of the data.

4. **<span style="color:blue"><u>Serializable</u></span>**: This level must prevent all possible anomalies, it provides a complete isolation between transactions.
    In practice, this is not totally true since the application has to retry transactions aborted because of impossibility of the DB to ensure serializability.

**<u>Read Uncommitted</u>** is not supported in PostgreSQL (no dirty reads allowed). It supports all other levels (with more restrictions on possible anomalies at each level).
It also uses **<u>Read Committed</u>** as the **default isolation level**.

#### Multiversion concurrency control.

PostgresSQL implements isolation through multiversion concurrency control (MVCC). A technique that mainly aims for this property:

**Readers never block writers, and writers never block readers.**

This idea is very similar to copy on write (COW) for standard data structures.

Imagine a tree like data structure. Write operations do not directly modify parts of the tree nodes but instead are copied and modified. 
Then when the write operation is finished, the copy is atomically swapped with the original data structure to allow readers to see the changes.

Take a look at the following diagram:

![mvcc diagram COW](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_copy_on_write.png)

COW allows us to serve multiple readers and one writer at the same time. When a writer is modifying the data structure, readers can still see the old versions
and a writer can copy necessary parts and proceed to modify them without getting blocked by readers.

You can see that multiple versions of the data can exist at the same time. What PostgresSQL does in MVCC has many resemblances with this example.

Without going a lot into implementation details, MVCC in PostgreSQL works in the same way:

1. **Readers don't block writers**, and **writers don't block readers**.

2. There could be multiple versions of a row at the same time. An read only operation in a transaction (at any isolation level) **can only see one of them**.

3. **Writers block writers**. Updating (but also deleting) a row necessitates acquiring a lock on that row. This does not block a reader querying previous versions of that row (look at diagram above). Lock is held until the end of the transaction and not just the end end for the update.

Let's stat discussing the second point. When a transaction starts, it will have visibility on data that was committed **before it has started**.

Let's not forget that:
- A transaction is a set of operations 
- The default isolation level in PostgreSQL is **Read Committed**. 

This means that an **operation** will see data committed before it started to run even if if was **not committed yet at the start of the transaction**. This, as we saw, causes the non repeatable read anomaly. Here is a very simple example that you can run to verify. 

Imagine you have these two transactions:

```sql
BEGIN;

SELECT * FROM users WHERE id = 1;
SELECT pg_sleep(2); -- wait for two seconds
SELECT * FROM users WHERE id = 1;

END;
```

```sql
BEGIN;
SELECT pg_sleep(1); -- wait for a second
UPDATE users SET emails_sent = emails_sent + 1 WHERE id = 1;
END;
```

Run them concurrently:

```sh
docker pull postgres:latest && \
docker run --name pg-test -e POSTGRES_USER=testuser -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=testdb -d -p 5432:5432 postgres:latest && \
sleep 5 && \
docker exec -i pg-test psql -U testuser -d testdb -c "CREATE TABLE users (id SERIAL PRIMARY KEY, emails_sent INT DEFAULT 0);" && \
docker exec -i pg-test psql -U testuser -d testdb -c "INSERT INTO users (emails_sent) VALUES (0);" && \
echo "Running two transactions concurrently" && \
# Running the first SQL transaction
docker exec pg-test psql -U testuser -d testdb -c "BEGIN; \
SELECT * FROM users WHERE id = 1; \
SELECT pg_sleep(2); \
SELECT * FROM users WHERE id = 1; \
END;" & \
# Running the second SQL transaction concurrently
docker exec pg-test psql -U testuser -d testdb -c "BEGIN; \
SELECT pg_sleep(1); \
UPDATE users SET emails_sent = emails_sent + 1 WHERE id = 1; \
END;" >/dev/null & \
# Wait for both background processes to finish
wait && \
# Check the final result of emails_sent
docker container stop pg-test > /dev/null && \
docker container rm pg-test > /dev/null
```

You should see the expected result:

```
 id | emails_sent
----+-------------
  1 |           0
----+-------------
```

And then (after the second transaction has committed):

```
 id | emails_sent
----+-------------
  1 |           1
----+-------------
```

But individual read only queries (operations) will have visibility on a single **snapshot** of the data, even if other concurrent transactions have committed since the start of the operation. 

If we go back to our teaser question in <span style="color:red"><u>example 3</u></span>.
If you run the following query:

```sql
SELECT * FROM students WHERE department = 'Computer Science';
```

It has a view on data that will not be altered by other concurrent committed transactions (updates, deletes or insertions). 
If `Bob` was in computer science, at the start of the operation, he will be in the result set as a computer science student even if he changed 
department to Mathematics by another committed transaction before the operation got to it.


Here is a concrete example. We start with a table containing 5 rows with `x=1` each.

- A first transaction starts first and will select all rows sleeping for a second using a cursor (to allow second transaction to execute) when reading every row.
- A second transaction starets **after** the first one and will update all rows to `x=2`.

Since the first transaction sees only one snapshot, we expect to see it's result to be rows with `x=1`

```sh
# Step 1: Pull and run the PostgreSQL container
docker pull postgres:latest && \
docker run --name pg-test -e POSTGRES_USER=testuser -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=testdb -d -p 5432:5432 postgres:latest && \
sleep 5 && \

# Step 2: Create the table and populate it with 5 rows
docker exec -i pg-test psql -U testuser -d testdb -c "CREATE TABLE my_table (id SERIAL PRIMARY KEY, x INT);" && \
docker exec -i pg-test psql -U testuser -d testdb -c "INSERT INTO my_table (x) VALUES (1), (1), (1), (1), (1);" && \
echo "Running two transactions concurrently" && \

# Step 3: First transaction to read rows with a delay
(docker exec pg-test psql -U testuser -d testdb -c "BEGIN; \
DECLARE my_cursor CURSOR FOR SELECT * FROM my_table; \
FETCH NEXT FROM my_cursor; SELECT pg_sleep(1); \
FETCH NEXT FROM my_cursor; SELECT pg_sleep(1); \
FETCH NEXT FROM my_cursor; SELECT pg_sleep(1); \
FETCH NEXT FROM my_cursor; SELECT pg_sleep(1); \
FETCH NEXT FROM my_cursor; \
CLOSE my_cursor; \
COMMIT;"; echo "finished selecting: " $(date)) & \

# Step 4: Second transaction to update all rows concurrently
(docker exec pg-test psql -U testuser -d testdb -c "BEGIN; \
SELECT pg_sleep(1); \
UPDATE my_table SET x = 2; \
COMMIT" >/dev/null; echo "finished update: " $(date)) & \

# Wait for both transactions to complete
wait && \

# Step 5: Check the final result
docker exec -i pg-test psql -U testuser -d testdb -c "SELECT * FROM my_table;" && \

# Cleanup
docker container stop pg-test > /dev/null && \
docker container rm pg-test > /dev/null
```

Even if the update transaction would be long ago finished, 
the first one will only keep seeing the first snapshot that it had access to when it started.

You should see that the cursor is always returning `x=1` for all rows even if the update transaction has finished.

```
finished update:  Wed Dec 11 08:36:52 PM CET 2024
finished selecting:  Wed Dec 11 08:36:55 PM CET 2024
```



The third point (**writers block writers**) allows us to answer two parts of teaser questions:

In <span style="color:red"><u>Example 2:</u></span>, we wanted to increment the number of emails sent to the user.

```sql
UPDATE users
SET emails_sent = emails_sent + 1
WHERE id = 1;
```

Since writers block writers, this transactions is safe to run concurrently with other transactions like this without any problem. No anomalies will occur. When trying to update 
the `emails_sent`, the row will be locked and all other transactions will wait until the lock is released.

You can run the following to check. It launches 100 parallel transactions to increment `emails_sent`:

```sh
docker pull postgres:latest && \
docker run --name pg-test -e POSTGRES_USER=testuser -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=testdb -d -p 5432:5432 postgres:latest && \
sleep 5 && \
docker exec -i pg-test psql -U testuser -d testdb -c "CREATE TABLE users (id SERIAL PRIMARY KEY, emails_sent INT DEFAULT 0);" && \
docker exec -i pg-test psql -U testuser -d testdb -c "INSERT INTO users (emails_sent) VALUES (0);" && \
echo "Incrementing emails_sent 100 times in parallel" && \
seq 1 100 | xargs -P100 -I{} docker exec -i pg-test psql -U testuser -d testdb -c "UPDATE users SET emails_sent = emails_sent + 1 WHERE id = 1;" > /dev/null && \
docker exec -i pg-test psql -U testuser -d testdb -c "SELECT emails_sent FROM users WHERE id = 1;" && \
docker container stop pg-test > /dev/null && \
docker container rm pg-test > /dev/null
```

The final result should always be 100 as expected. Concurrently incrementing emails_sent in this way is safe.

Now then what is the difference with the following query?

```sql
BEGIN;
-- you can't directly assign variables in PG this way but assume this is done in code
current_emails_sent := (SELECT emails_sent FROM users WHERE id = 1);

UPDATE users
SET emails_sent = current_emails_sent + 1
WHERE id = 1;

END;
```

This is a standard **use-after-check** bug that is very prone to race conditions in concurrent setups.
Concurrent transactions can read a old version of the row while another transaction is updating it.
They then try to update it based on the **now** stale data, leading to inconsistent results.

What happens if two transactions try to run concurrently? Try the following:

```sh
docker pull postgres:latest && \
docker run --name pg-test -e POSTGRES_USER=testuser -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=testdb -d -p 5432:5432 postgres:latest && \
sleep 5 && \
docker exec -i pg-test psql -U testuser -d testdb -c "CREATE TABLE users (id SERIAL PRIMARY KEY, emails_sent INT DEFAULT 0);" && \
docker exec -i pg-test psql -U testuser -d testdb -c "INSERT INTO users (emails_sent) VALUES (0);" && \
echo "Incrementing emails_sent 100 times in parallel using DO block in transaction" && \
seq 1 100 | xargs -P100 -I{} docker exec -i pg-test psql -U testuser -d testdb -c "DO \$\$ 
DECLARE 
  current_emails_sent INT;
BEGIN 
  -- Get the current value of emails_sent
  SELECT emails_sent INTO current_emails_sent FROM users WHERE id = 1;
  
  -- Increment the emails_sent
  UPDATE users SET emails_sent = current_emails_sent + 1 WHERE id = 1;
END;
\$\$;" > /dev/null && \
docker exec -i pg-test psql -U testuser -d testdb -c "SELECT emails_sent FROM users WHERE id = 1;" && \
docker container stop pg-test > /dev/null && \
docker container rm pg-test > /dev/null
```

Depends on the run, but you probably will get an invalid result:

```
 emails_sent
-------------
          33
-------------
```

In this specific case, the first version is cleaner and works fine, but in practice we might have to do with more complicated logic.
Imagine for example you have a stock of some product, a user wants to buy 3 units.
You want to check in the DB if there is enough before validating the purchase. 

Fortunately, PostgresSQL provides a way to lock a row from other writers using a `FOR UPDATE` clause.

This clause locks rows returned by `SELECT` as if it was updating them and thus prevents other transactions from locking, updating or deleting these rows until the transaction ends.
Conversely it will block when trying to lock a row that is already locked by another transaction.

`FOR UPDATE` is pretty expressive, it's you telling the DB: I am going to be locking this row because I will be updating it potentially.

To apply this to our example, we can use the following query:

```sql
BEGIN;
-- you can't directly assign variables in PG this way but assume this is done in code
current_emails_sent := (SELECT emails_sent FROM users WHERE id = 1 FOR UPDATE);
                                             -- Difference here ---^
UPDATE users
SET emails_sent = current_emails_sent + 1
WHERE id = 1;

END;
```

Try this now:

```sh
docker pull postgres:latest && \
docker run --name pg-test -e POSTGRES_USER=testuser -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=testdb -d -p 5432:5432 postgres:latest && \
sleep 5 && \
docker exec -i pg-test psql -U testuser -d testdb -c "CREATE TABLE users (id SERIAL PRIMARY KEY, emails_sent INT DEFAULT 0);" && \
docker exec -i pg-test psql -U testuser -d testdb -c "INSERT INTO users (emails_sent) VALUES (0);" && \
echo "Incrementing emails_sent 100 times in parallel using DO block in transaction" && \
seq 1 100 | xargs -P100 -I{} docker exec -i pg-test psql -U testuser -d testdb -c "DO \$\$ 
DECLARE 
  current_emails_sent INT;
BEGIN 
  -- Get the current value of emails_sent
  SELECT emails_sent INTO current_emails_sent FROM users WHERE id = 1 FOR UPDATE;
                                                -- Difference here ---^
  -- Increment the emails_sent
  UPDATE users SET emails_sent = current_emails_sent + 1 WHERE id = 1;
END;
\$\$;" > /dev/null && \
docker exec -i pg-test psql -U testuser -d testdb -c "SELECT emails_sent FROM users WHERE id = 1;" && \
docker container stop pg-test > /dev/null && \
docker container rm pg-test > /dev/null
```

You should get the correct results now:

```
 emails_sent
-------------
         100
-------------
```


Let's go back to <span style="color:red"><u>example 4</u></span> now. We wanted to update two rows at the same time.

Each transaction will update both rows, and thus taking a lock on one row and then the other. But in which order? 
We have no way of knowing which order the rows will be locked.

Imagine transaction 1 tries to update row with `id=1` so it takes a lock on it, transaction 2 takes a lock on row with `id=2`. 
Transaction 1 will now want to lock on row with `id=2`, it's already locked so it will wait for the lock to be released. But transaction 2 is already holding a lock on row with `id=2` and waiting for the lock on row with `id=1`, both transactions are waiting for each other => **DEADLOCK**

Let's first try to run the following transactions concurrently:

```sql
BEGIN;

UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 1; -- fist 1
SELECT pg_sleep(1); -- wait for a second
UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 2; -- then 2

END;
```

```sql
BEGIN;

Update games SET nb_games = nb_games + 1 WHERE player_id = 2; -- first 2
SELECT pg_sleep(1); -- wait for a second
Update games SET nb_games = nb_games + 1 WHERE player_id = 1; -- then 1

END;
```

Try the following (and remember that a lock on row lives until the end of the transaction):

```sh
docker pull postgres:latest && \
docker run --name pg-test -e POSTGRES_USER=testuser -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=testdb -d -p 5432:5432 postgres:latest && \
sleep 5 && \
docker exec -i pg-test psql -U testuser -d testdb -c "CREATE TABLE games (player_id INT PRIMARY KEY, nb_games INT DEFAULT 0);" && \
docker exec -i pg-test psql -U testuser -d testdb -c "INSERT INTO games (player_id, nb_games) VALUES (1, 0), (2, 0);" && \
printf "BEGIN; UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 1; SELECT pg_sleep(1); UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 2; COMMIT;\nBEGIN; UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 2; SELECT pg_sleep(1); UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 1; COMMIT;\n" | \
xargs -P2 -I{} docker exec -i pg-test psql -U testuser -d testdb -c "{}" ; \
docker exec -i pg-test psql -U testuser -d testdb -c "SELECT * FROM games;" && \
docker container stop pg-test > /dev/null && \
docker container rm pg-test > /dev/null
```

You will get an error message like this:

```
ERROR:  deadlock detected
DETAIL:  Process 94 waits for ShareLock on transaction 743; blocked by process 95.
Process 95 waits for ShareLock on transaction 742; blocked by process 94.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "games"
```

Pg detected a circular lock dependency and chose to abort one of them to break the deadlock. Only one of the two transactions will be committed.

```
 player_id | nb_games
-----------+----------
         1 |        1
         2 |        1
----------------------
```

A less apparent problem is the one in <span style="color:red"><u>example 4</u></span>:

```sql
BEGIN;

UPDATE games SET nb_games = nb_games + 1 WHERE player_id IN (1, 2);

END;
```

Unfortunately, the `IN` clause does not guarantee that the locks will be taken in the same order. So having concurrent transactions of this form can lead
to deadlock situations and some transactions will be aborted, so you should be ready to retry them if (and when) this happens.

To solve the first case, we can just update rows in the same order for all transactions: 

```sql
BEGIN;

UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 1;
UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 2;

END;
```

What about the second case? We need to ensure that locks are acquired in the same order for all transactions.

As we saw above, we can take a lock on row with the **intension** of updating (but not necessarily updating it) it by using the `FOR UPDATE` clause.

We can do the following to remove the risk of deadlocks:

```sql
BEGIN;

UPDATE games SET nb_games = nb_games + 1 WHERE player_id IN (
    SELECT player_id FROM games WHERE player_id IN (1, 2) ORDER BY player_id FOR UPDATE
);

END;
```

This ensures all concurrent transactions will take a lock on the same rows in the same order and thus avoid deadlocks.

This only works if the `player_id` is immutable, which is generally a good assumption for identifier columns. Otherwise it would not be deadlock safe.
Locking happens after the `ORDER BY` clause. If a concurrent transaction updates rows after the `ORDER BY` and before locking, and 
this update changes sort order, locking will happens out of order (will be based on old order) and the transaction can deadlock with another transaction 
that will lock the rows in the new order.


Big updates are not generally a good idea: they hurt performance, bloat the table (since we have to keep old version in an MVCC strategy) and 
increase contention (a lock is held during the entire transaction lifetime). So try to run updates in batches and always be ready to retry aborted transactions due to deadlocks.

To resume what we have seen before jumping into `Repeatable Read` and `Serializable` isolation levels:

1. Transactions can cause anomalies when run concurrently.
2. PostgresSQL implements MVCC to isolate transactions and provides three levels of isolation: `Read Committed`, `Repeatable Read` and `Serializable`.
3. At `Read Committed` level, transactions can see only committed data. Queries (operations) can see snapshots of data committed before they are run even if it was committed after the start of the transaction (leading to non repeatable read anomalies).
4. Writers always block writers. Updating a row (or deleting it) necessitates acquiring a lock on that row, but also selecting a row `FOR UPDATE`.
5. Read only queries inside a transactions have a single snapshot of the data and it's never altered by other concurrent committed transactions.
6. Deadlocks can happen when two transactions try to lock the same rows in different orders. PostgresSQL detects them automatically and aborts necessary transactions to allow the system to move forward. If you are not sure your transactions are deadlock free, prepare to retry them.

