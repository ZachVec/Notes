# HOMEWORK #1 - SQL

## Overview

The homework #1 is quiet simple if you are already familiar with SQL and it's advanced usages like Window Function and Common Table Expression, aka CTE. And it could also be pretty struggling if you are a newbie to the SQL, i.e., you have never learn anything about SQLs. In that case, you may wish to find some starter course to learn some basics.

The specification of homework1 can be found [here](https://15445.courses.cs.cmu.edu/fall2020/homework1/). Before working on the homework, you may need to check if your sqlite3 DBMS has been properly installed and database [musicbrainz](https://15445.courses.cs.cmu.edu/fall2020/files/musicbrainz-cmudb2020.db.gz) has been downloaded. Just follow the instructions in guidance and you will be fine.

Note: the database can be extremely hard to download if you are in countries or regions behind a firewall.

## Summary

**CAST function**

`CAST` function casts variables to other type with the syntax like:

```sql
SELECT CAST(year AS VARCHAR) || some_strings   -- if you are going to concatenate an INT with string
FROM some_table;
```

**Aggregates**

Aggregate function consists of `COUNT()`, `AVG()`, `MIN()`, `MAX()` and `SUM()`. These functions will produce only one aggregated results for each group or the whole relation, i.e., table, if the group is not specified.

Additionally, if you would like to filer the results using the value provided by aggregate functions, use `HAVING` clause instead of `WHERE` clause.

**Window Functions**

Apart from the functions mentioned in **Aggregates**, **Window Functions** has `ROW_NUMBER()`, `RANK()` and `DENSE_RANK()` (`DENSE_RANK()` is not covered in neither lecture nor note and you can just leave this behind.)

The aggregate functions almost do the same thing with the window functions, except that the aggregate functions produce one tuple per group, whereas the window functions attached the aggregated result to all the tuples in the relation.

Given a table named `grade` like the following: (`cid` stands for course id, and `sid` stands for student id)

```
+---------+---------+-----------+
|   cid   |   sid   |    gpa    |
+---------+---------+-----------+
|  15213  |    1    |     A     |
+---------+---------+-----------+
|  15213  |    2    |     B     |
+---------+---------+-----------+
|  15445  |    1    |     B     |
+---------+---------+-----------+
|  15445  |    3    |     C     |
+---------+---------+-----------+
```

If we use `Aggregate function` like the `sql` below:

```sql
SELECT cid, MIN(gpa)
FROM grade
GROUP BY cid
ORDER BY cid;
```

We will get the following result

```
+---------+---------+
|   cid   |   gpa   |
+---------+---------+
|  15213  |    A    |
+---------+---------+
|  15445  |    B    |
+---------+---------+
```

Whereas we use `Window function`

```sql
SELECT cid, MIN(gpa) OVER(
	GROUP BY cid
	ORDER BY cid
)
FROM grade;
```

We will have

```
+---------+-----------+
|   cid   |    gpa    |
+---------+-----------+
|  15213  |     A     |
+---------+-----------+
|  15213  |     A     |
+---------+-----------+
|  15445  |     B     |
+---------+-----------+
|  15445  |     B     |
+---------+-----------+
```

---

**Nested Queries**

As concluded in the provided [note](https://15445.courses.cs.cmu.edu/fall2020/notes/02-advancedsql.pdf), nested queries can show up in almost any part of query. But in my practice, nested queries never appeared in `SELECT` Output targets, can be replaced with `WITH` clause and be more readable. The only part the nested query shows up is the `WHERE` clause.

In my practice, if you are gonna select those with certain criterion like `id = some_kinda_id`, the nested query could help you out with that without using more `INNER JOIN`s, which would significantly harm the performance.

For example, queries using nested query:

```sql
SELECT name
FROM artist
WHERE type = (SELECT id FROM artist_type WHERE name = 'Person');
```

can be faster than the queries instead using join:

```sql
SELECT name
FROM artist
	INNER JOIN artist_type ON artist.type = artist_type.id
WHERE artist_type = 'Person';
```

The difference can be hard to tell if the number of tuples is not that many, but it can be much more apparent in more complicated queries.

