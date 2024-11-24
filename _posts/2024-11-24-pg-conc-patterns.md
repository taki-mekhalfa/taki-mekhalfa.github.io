---
layout: post
title: "Database Concurrency Patterns For Developers"
categories: misc
---

I have been developing applications interacting with databases for many years now and I have always taken concurrency for granted. I always looked at the database as a black box and I never had to think about concurrency issues assuming that the database would handle it for me. Especially when I was working with relational OLTP databases (PostgreSQL mainly) where there was more business logic involved.

This is due to the fact the my interacions with the DB were mainly for CRUD-like applications: inserting, updating, deleting and selecting data generally for individual records. This has always worked and did not bother me much.

At one point, I had to work on a problem to implement a business logic on top of the database that required a careful handling of concurrency and I could not reason about it or if my approach was correct. I was just jumping from a stack overflow post to another one, not sure if I understood the problem and only trying to find a way to take locks as I was mentally drawing a parallel between concurrency at the database level and general tools and patterns for concurrent programming.

I decided that it was best to take the time and effort to understand different guarantees and tools provided by the database and how they work and it helped me a lot to reason about the correctness of my queries and allowed me to beter understand the trade-offs of different approaches.

In this post, I will share some what I have learned and encourage you to dive deepr into the topic if needed.

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
-- you can't directly assign variables in PG but assume this is done in code
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

Here is a more subtle example: let's say your API receives queries to increment the number of games of two players. You receive two queries with the same players. You open two transactions and increment the number of games for each player but the order in the two transactions is not the same. So you will execute the following queries in parallel:

```sql
BEGIN;

UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 1;
UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 2;

END;
```

and this one:

```sql
BEGIN;

UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 2;
UPDATE games SET nb_games = nb_games + 1 WHERE player_id = 1;

END;
```

What bad things can happen?

#### <u>[Why is concurrency control important at the database level?]</u>