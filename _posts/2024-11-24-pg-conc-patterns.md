---
layout: post
title: "Database Concurrency Patterns For Developers"
categories: misc
---

I have been developing applications interacting with databases for many years now and I have always taken concurrency for granted. I always looked at the database as a black box and I never had to think about concurrency issues assuming that the database would handle it for me. Especially when I was working with relational OLTP databases where there was more business logic involved.

This is due to the fact that my interactions with the DB were mainly for CRUD-like applications: inserts, stateless updates, deletes and reading data back. Generally for individual records. This has always worked and did not bother me much.

At one point, I had to work on a problem to implement a business logic on top of the database that required a careful handling of concurrency and I could not reason about it or if my approach was correct. I was just jumping from a stack overflow post to another one, not sure if I understood the problem and only trying to find a way to take locks as I was mentally drawing a parallel between concurrency at the database level and general tools and patterns for concurrent programming.

I decided that it was best to take the time and effort to understand different guarantees and tools provided by the database and how they work and it helped me a lot to reason about the correctness of my queries and allowed me to better understand the trade-offs of different approaches.

In this post, I will share some what I have learned and encourage you to dive deeper into the topic if needed.

⚠️ All information below is valid for PostgreSQL and might not be applicable to other databases.

#### <u>[Teaser questions to get you started]</u>

If you have doubts or difficulties answering them, then this blog post if for you:

##### <u>Example 1:</u>

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

##### <u>Example 2:</u>

Now imagine you have a counter that is incremented to account for the numbers of emails sent to the user for example. 

Would it work if you have many concurrent queries trying to increment the counter at the same time?

```sql
UPDATE users
SET emails_sent = emails_sent + 1
WHERE id = 1;
```

what's is the difference with the following query?

```sql
BEGIN;

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

##### <u>Example 3:</u>

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

Imagine another concurrent query is running and it will update the department of `Bob` to `Mathematics` before the first query gets to the last row. What would happen? Would `Bob` be in the result set or not?

##### <u>Example 4:</u>

Here is a more subtle example: let's say your API receives queries to increment the number of games of two players. You receive two queries with the same players. You open two transactions and increment the number of games for each player. So you will execute the following queries in parallel:

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

What bad things can happen?

#### <u>[Why is concurrency control important at the database level?]</u>

It's all about <u>consistency</u>. What is consistency? It means that the data is always in a valid state. What does it mean for the data to be valid? Well it depends on your application and the business logic you are implementing.

Some consistency guarantees can (and should) be expressed in the form of constraints at the database level. For example, you can enforce that a column can be used as an identifier for individual records by being unique and not null by telling the database that it is a primary key. It's the job of the database to enforce this constraint and not allow you to violate it and you can be sure that the data is always valid from this point of view.

Not all consistency guarantees can be easily or even possibly expressed at the database level in the form of constraints. So the burden is on the application (and thus you) to ensure that the data is always valid. For example, booking a room should only be possible two weeks in advance except for clients with a premium subscription. You have to programmatically make sure this is enforced when receiving a booking request.

We can make sure data is always in a valid state if we apply operations that take it from a valid state to a another valid state as shown in the following diagram:

![consistency diagram](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_transistion.png)

Sometimes, some operations need to break consistency momentarily to implement some business logic. For example, you debit 10$ from one account and credit it to another. You can see that the period of time after the debit and before crediting the second account is a period of time where the data is not in a valid state (10$ just disappeared). What happens if there is a failure just before crediting the second account? Well, consistency is broken and the gate to hell is wide open.

![consistency diagram bad state](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_transistion_bad_state.png)

#### We need a transaction!

To avoid the previous situation, the database provides a way to group operations together and make sure they are always ALL executed or NONE of them. If none of them are executed we will stay at the same valid state as before, if they all are executed, we will end up in a valid state (assuming that this is implied by rules on the DB or your application logic). This is called a **transaction**. Transactions are then a group of operations that are executed as a single unit of work (ATOMIC) and can only take the data from a CONSISTENT state to a another CONSISTENT state. These are your first two letters of <u>AC</u>ID.

![consistency diagram transaction](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_transistion_tx.png)

#### Everything is a transaction!

PostgreSQL treats each statement as a transaction. If you issue a statement, it will be wrapped in a transaction and committed if successful or rolled back if an error occurs. If we go back to our teaser questions, then these two are actually equivalent:

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

The database needs to handle high throughput of transactions. This means that it needs to be able to execute transactions concurrently. What's the problem with that? A set of operations that will leave the DB in a valid state if executed serially will leave it in an invalid state if executed concurrently. 

Take the following example: A user has 100$ in their account. Two concurrent transactions try to debit 10$ from the account and credit it to another account. They both succeed and both try to update the balance of the user to 90$ (100$ - 10$). This is an inconsistent state as 10$ was created from thin air and the user should actually have had 80$ left in their account.

You can see that if both transactions were run serially, first one and then the second one (or vice versa), the balance would be consistent. 

This does not always happen. Imagine one of the transactions was only reading the balance and the other one was updating it or that both transactions were reading and writing different records. Those could potentially be run concurrently with no problem. The goal of the database is then to run operations as consistently as possible without leading to inconsistent states of the data. 

A valid concurrent execution of a set of operations is valid if and only if the final state corresponds to the outcome of one of the possible serial executions of the operations (because we know that a serial execution of transactions leave the data in a valid state).

Here is an example: Imagine you have two transactions, one want to add 10$ to the balance and another one wants to increase it by 10% as follows:

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

* 121$ (if the first transaction was executed first and then the second one)
* 120$ (if the second transaction was executed first and then the first one)

All other outcomes are invalid. 110$ is not valid if both read 100$ at the same time and then update. 

Invalid states are called **anomalies**.

#### Isolation!

We come to our third letter of AC<u>I</u>D. Isolation is the ability of the database to execute transactions concurrently without violating consistency, i.e. avoid anomalies.

Here are some example anomalies:

#### <u>Lost update anomaly:</u>

Going back to our example where two transactions tried to debit 10$ from the user's account. One transaction updated the final balance to 90$ but the other one was not aware of that and continued thinking that the balance was still 100$, so it updated it to 90$ too. The update from the first transaction was lost.

![consistency diagram lost update](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_lost_update.png)


#### <u>Dirty read anomaly:</u>

Dirty read anomalies happen when a transaction reads data written by another concurrent transaction that was not committed.
Imagine a transaction wants adds 10$ to two accounts, it credits the first account but fails to credit the second one and thus it rollbacks.
A second concurrent transaction checks the balance of the first account, it sees 10$ and allows them to buy something before the first transaction rollbacks.
This is an anomaly because the first account never really had 10$ in it at the time of the second transaction.


![consistency diagram dirty read](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_dirty_read.png)

You should know that this anomaly is **not permitted in PostgreSQL**, and you don't have to worry about it at all.
The effects of a transaction can not be partially observed by another transaction while it's not yet committed. This is generally a consequence of atomicity.


From the point of view of other concurrent transactions, a transactions happens as a whole instantaneously (if not rolled back).
This means that when you write a transaction, you can be sure that others won't observe changes only after the transaction is committed.

#### <u>Non-repeatable read anomaly:</u>

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

This invalid state would happen if the second transactions added an employee in the Computer department with a salary of 200.

![consistency diagram non repeatable read](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/2-pg-conc-patterns/consistency_non_repeatable_read.png)
