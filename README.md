# sql-blog-paging (github.com/antonio-alexander/sql-blog-paging)

This is a repo that tries to come up with some practical solutions for paging. The premise is that in each table or database, there may be so much data that its unreasonable to query it at once. The easiest example is the results of a web search, at the bottom there may be a lot of results, but you only see the first ten (e.g. page 1 of 10). Some of the things that are expected (i.e. hidden requirements) are that the results remain consistent with the context of the search: if you don't change your search parameters, the returned results should be the same; and that the results should come back relatively quickly.

> In case you're really in the dark about pagination, pagination allows you to query a semi-static set of data and separate it in pages such that you only query a chunk of data because you can only display a limited amount of data of the resources/latency associated with querying all the data at once is unreasonable.

Although there's some application related code/processes that has to do with paging, my goal with this repo is to focus on the sql. So, there will be a lot of effort spent on query optimization and query result consistency. For purposes of testing, I used the [employees test database](https://github.com/datacharmer/test_db) since it's a common standard for learning and optimizing databases (not just for paging).

The point of this repo is to do two things which are related, but very much two different goals: (1) how to do pagination and (2) how to optimize queries that use pagination through indexes. At a high level, these are the use cases I want to figure out:

- How to do offset based pagination
- How to do token-based pagination
- How to maintain data consistency between different pages of the same query/parameters
- Basic performance differences between offset/token pagination

A distinction I want to make that may help you understand the point of view of most of these solutions is that they are meant to be used by application APIs (that are probably asynchronous) versus someone with an interactive session with the database. There is __some__ parity between those kinds of APIs and interactive sessions, but some things are difficult.

## Bibliography

- [https://betterprogramming.pub/why-token-based-pagination-performs-better-than-offset-based-465e1139bb33](https://betterprogramming.pub/why-token-based-pagination-performs-better-than-offset-based-465e1139bb33)
- [https://robconery.medium.com/creating-a-full-text-search-engine-in-postgresql-2022-7b6ed6e81cae](https://robconery.medium.com/creating-a-full-text-search-engine-in-postgresql-2022-7b6ed6e81cae)
- [https://betterprogramming.pub/you-are-doing-sql-pagination-wrong-739a700acbd0](https://betterprogramming.pub/you-are-doing-sql-pagination-wrong-739a700acbd0)
- [https://www.sentinelone.com/blog/how-to-measure-mysql-query-time/](https://www.sentinelone.com/blog/how-to-measure-mysql-query-time/)
- [https://www.sitepoint.com/using-explain-to-write-better-mysql-queries/](https://www.sitepoint.com/using-explain-to-write-better-mysql-queries/)
- [https://medium.com/swlh/how-to-implement-cursor-pagination-like-a-pro-513140b65f32](https://medium.com/swlh/how-to-implement-cursor-pagination-like-a-pro-513140b65f32)
- [https://developer.squareup.com/blog/tips-and-tricks-for-api-pagination/](https://developer.squareup.com/blog/tips-and-tricks-for-api-pagination/)
- [https://dev.mysql.com/doc/refman/8.0/en/table-scan-avoidance.html](https://dev.mysql.com/doc/refman/8.0/en/table-scan-avoidance.html)

## Getting Started

To try most of what's in this document, you'll need to load the test data (employees) from [https://github.com/datacharmer/test_db](https://github.com/datacharmer/test_db); This repo is configured with a submodule holding that repository and the [docker-compose.yml](./docker-compose.yml) is configured with a volume to load that submodule. The data can be loaded using the following commands:

```sh
docker compose up -d
docker exec -it mysql /bin/ash
cd /tmp/test_cd test_db
mysql -u root -p < ./employees.sql
```

For more information, you can look at the [README.md](https://github.com/datacharmer/test_db/blob/master/README.md)

## Limit/Offset Based Pagination

Limit/Offset based pagination can be considered the least complex solution for pagination, it works off the idea that if you have a query that returns 100 items, you can divide that into one or more pages by using the offset and limit function (give me items 0-10, then give me items 11-20 etc.); it also allows you to be able to determine the total number of pages, the total number of rows as well as the ability to calculate the limit/offset for a given page.

For this example, I'm going to try to expose the weaknesses of offset based pagination both in terms of data consistency, but also in terms of overall performance. I'm going to be using this query for this example:

```sql
SELECT emp_no, first_name, last_name FROM employees WHERE gender='M' ORDER BY emp_no ASC;
```

We can find the total number of rows our query would generate with the following query:

```sql
SELECT count(*) AS total FROM employees WHERE gender='M';
```

```log
+--------+
| total  |
+--------+
| 179973 |
+--------+
1 row in set (0.086 sec)
```

We can calculate the number of pages by dividing the total by the number of rows we want per page:

- with 10 rows per page, we have 17998 pages
- with 100 rows per page, we have 1800 pages

The limit is static and will be the number of rows per page, while the offset will be a function of the page number:

- with 10 rows per page, page 2, would have an offset of 10 (1*10) and a limit of 10
- with 10 rows per page, page 1000, would have an offset of 10000 (1000*10) and a limit of 10
- with 100 rows per page, page 25, would have an offset of 2500 (25*100) and a limit of 100

We can perform a paginated query by adding the limit and offset keywords to the above query like so:

```sql
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 20;
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 0;
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 5;
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 10;
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 15;
```

Below we're going to attempt to execute each query; I'll point out some notes after.

```log
MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 20;
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10001 | Georgi     | Facello     | M      |
|  10003 | Parto      | Bamford     | M      |
|  10004 | Chirstian  | Koblick     | M      |
|  10005 | Kyoichi    | Maliniak    | M      |
|  10008 | Saniya     | Kalloufi    | M      |
|  10012 | Patricio   | Bridgland   | M      |
|  10013 | Eberhardt  | Terkki      | M      |
|  10014 | Berni      | Genin       | M      |
|  10015 | Guoxiang   | Nooteboom   | M      |
|  10016 | Kazuhito   | Cappelletti | M      |
|  10019 | Lillian    | Haddadi     | M      |
|  10020 | Mayuko     | Warwick     | M      |
|  10021 | Ramzi      | Erde        | M      |
|  10022 | Shahaf     | Famili      | M      |
|  10025 | Prasadram  | Heyers      | M      |
|  10026 | Yongqiao   | Berztiss    | M      |
|  10028 | Domenick   | Tempesti    | M      |
|  10029 | Otmar      | Herbst      | M      |
|  10030 | Elvis      | Demeyer     | M      |
|  10031 | Karsten    | Joslin      | M      |
+--------+------------+-------------+--------+
20 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 0;
+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10001 | Georgi     | Facello   | M      |
|  10003 | Parto      | Bamford   | M      |
|  10004 | Chirstian  | Koblick   | M      |
|  10005 | Kyoichi    | Maliniak  | M      |
|  10008 | Saniya     | Kalloufi  | M      |
+--------+------------+-----------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 5;
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10012 | Patricio   | Bridgland   | M      |
|  10013 | Eberhardt  | Terkki      | M      |
|  10014 | Berni      | Genin       | M      |
|  10015 | Guoxiang   | Nooteboom   | M      |
|  10016 | Kazuhito   | Cappelletti | M      |
+--------+------------+-------------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 10;
+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10019 | Lillian    | Haddadi   | M      |
|  10020 | Mayuko     | Warwick   | M      |
|  10021 | Ramzi      | Erde      | M      |
|  10022 | Shahaf     | Famili    | M      |
|  10025 | Prasadram  | Heyers    | M      |
+--------+------------+-----------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 15;
+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10026 | Yongqiao   | Berztiss  | M      |
|  10028 | Domenick   | Tempesti  | M      |
|  10029 | Otmar      | Herbst    | M      |
|  10030 | Elvis      | Demeyer   | M      |
|  10031 | Karsten    | Joslin    | M      |
+--------+------------+-----------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> 
```

We can determine that data consistency is maintained: the results of the second and third query match the results of the first query. A situation where we wouldn't be able to maintain data consistency is if an employee was added where the employee number was less than the initial employee number (because we're sorting by employee number, it would be added to the "beginning" of the data set).

> This is an edge case, but you must keep in mind that when you're doing paging _in this way_ you're reading a moving target. It's a bit unreasonable to do this within a transaction and as a result it's possible for data to be inserted between reading pages.

```sql
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 20;
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 0;
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 5;
INSERT INTO employees(emp_no,birth_date,first_name,last_name,gender,hire_date) values('10000', '2022-10-02', 'John', 'Smith', 'M', '2022-10-02');
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 10;
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 15;
DELETE FROM employees WHERE emp_no='10000';
```

```log
MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 20;
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10001 | Georgi     | Facello     | M      |
|  10003 | Parto      | Bamford     | M      |
|  10004 | Chirstian  | Koblick     | M      |
|  10005 | Kyoichi    | Maliniak    | M      |
|  10008 | Saniya     | Kalloufi    | M      |
|  10012 | Patricio   | Bridgland   | M      |
|  10013 | Eberhardt  | Terkki      | M      |
|  10014 | Berni      | Genin       | M      |
|  10015 | Guoxiang   | Nooteboom   | M      |
|  10016 | Kazuhito   | Cappelletti | M      |
|  10019 | Lillian    | Haddadi     | M      |
|  10020 | Mayuko     | Warwick     | M      |
|  10021 | Ramzi      | Erde        | M      |
|  10022 | Shahaf     | Famili      | M      |
|  10025 | Prasadram  | Heyers      | M      |
|  10026 | Yongqiao   | Berztiss    | M      |
|  10028 | Domenick   | Tempesti    | M      |
|  10029 | Otmar      | Herbst      | M      |
|  10030 | Elvis      | Demeyer     | M      |
|  10031 | Karsten    | Joslin      | M      |
+--------+------------+-------------+--------+
20 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 0;
+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10001 | Georgi     | Facello   | M      |
|  10003 | Parto      | Bamford   | M      |
|  10004 | Chirstian  | Koblick   | M      |
|  10005 | Kyoichi    | Maliniak  | M      |
|  10008 | Saniya     | Kalloufi  | M      |
+--------+------------+-----------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 5;
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10012 | Patricio   | Bridgland   | M      |
|  10013 | Eberhardt  | Terkki      | M      |
|  10014 | Berni      | Genin       | M      |
|  10015 | Guoxiang   | Nooteboom   | M      |
|  10016 | Kazuhito   | Cappelletti | M      |
+--------+------------+-------------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> INSERT INTO employees(emp_no,birth_date,first_name,last_name,gender,hire_date) values('10000', '2022-10-02', 'John', 'Smith', 'M', '2022-10-02');
Query OK, 1 row affected (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 10;
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10016 | Kazuhito   | Cappelletti | M      |
|  10019 | Lillian    | Haddadi     | M      |
|  10020 | Mayuko     | Warwick     | M      |
|  10021 | Ramzi      | Erde        | M      |
|  10022 | Shahaf     | Famili      | M      |
+--------+------------+-------------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5 OFFSET 15;
+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10025 | Prasadram  | Heyers    | M      |
|  10026 | Yongqiao   | Berztiss  | M      |
|  10028 | Domenick   | Tempesti  | M      |
|  10029 | Otmar      | Herbst    | M      |
|  10030 | Elvis      | Demeyer   | M      |
+--------+------------+-----------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> DELETE FROM employees WHERE emp_no='10000';
Query OK, 1 row affected (0.005 sec)
```

Looking at the above queries, we can see that that the third page is "wrong", the employee Kazuhito Cappelletti, is at the end of the second page and the beginning of the third page, where the expectation would be that he would ONLY be on the second page, and that the third page would start with Lillian. It really depends on the lifetime of your data, how often its mutated and what the result would be for someone reading the data. In practice it's probably not a big deal, but here's one scenario that could be problematic:

> If you're reading/listing rows to edit a row, if your list contains stale data, it's possible that you would make those "write" decisions using the stale data and squash changes made in the interim. Two ways to solve this problem is to (1) make your update depend on an update audit field (which must be updated atomically) to determine if someone else has modified the data before your last read or (2) lock the row for writing such that no one else can modify it until you've modified it. SQL cursors provide this functionality.

Modifying the table to have an immutable, sequential and unique field such as a create_date or autonumber would provide a _initial_ sorting constraint that would enable __cursors__; for a given column used as a cursor, if the field isn't unique, it is difficult to guarantee that the page will provide the same results for the same query.

> This is a solvable problem, but it means that you must create a constraint that's not modifiable by your API (i.e. Human Resources). For example, you could add a constraint to sort by hire date, but hire date doesn't guarantee that a _new_ employee can't be added in-between your query; what if there was a merger and senior staff was maintained and for the purposes of this database, their hire dates in the old company had to be maintained (e.g. benefits); then your query could become inconsistent. That's a one-time issue, so not super practical, but depending on what data is in the table, it's purpose may have situations like that often.

In short, the __big__ issue with limit/offset queries is the offset, the offset makes the query linear rather than consistent; the bigger the offset, the longer the query will take. On the upside, with offset/limit pagination, you can randomly query any page if you know what the total is (I'd imagine this is an uncommon need, but otherwise a useful feature).

## Token/Cursor Based Pagination

Token (or more accurately cursor) based pagination is functionally like offset based pagination, but it provides a reference point (rather than just the beginning) for the query which makes the query more consistent in terms of performance no matter how big the data set is. Offset/Limit based pagination generally has a linear performance hit, the more data, the further the page, the longer it'll take to query the data. The biggest difference API-wise between the token/cursor-based pagination and the offset/limit based pagination is that the API provides you with two "cursors" that can be used to query the next or previous page.

Taking the previous query used in [limit/offset based pagination](#limitoffset-based-pagination):

```sql
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC;
```

For the purposes of this section, we'll use the employee id (emp_no) as the cursor.

> The cursor should be a column that won't change during the lifetime of a query (ideally the lifetime of the table) and that can be added as a reasonable constraint (i.e. can be easily compared in an open-ended way like greater than).

As stated earlier, the main difference between offset and token/cursor-based pagination is that instead of the offset, you use the cursor. We can modify the queries used above to the following (for demo purposes, I’m showing moving forward and backward through the pages):  

```sql
-- full data set
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 20; 
-- page 1 (no cursors) (cursor_next=10008)
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5; 
-- page 1->2 (cursor_next=10016, cursor_previous=10012)
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no > 10008 ORDER BY emp_no DESC LIMIT 5;
-- page 2->3 (cursor_next=10025, cursor_previous=10019)
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no > 10016 ORDER BY emp_no ASC LIMIT 5;
-- page 3->4 (cursor_previous=10026) 
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no > 10025 ORDER BY emp_no ASC LIMIT 5;
-- page 3<-4 (cursor_next=10025, cursor_previous=10019)
SELECT emp_no, first_name, last_name, gender FROM (
    SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no < 10026 ORDER BY emp_no DESC LIMIT 5
) AS backwards ORDER by backwards.emp_no ASC; 
-- page 2<-3 (cursor_next=10016, cursor_previous=10012)
SELECT emp_no, first_name, last_name, gender FROM (
    SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no < 10019 ORDER BY emp_no DESC LIMIT 5
) AS backwards ORDER by backwards.emp_no ASC; 
-- page 1<-2 (cursor_next=10008)
SELECT emp_no, first_name, last_name, gender FROM (
    SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no < 10012 ORDER BY emp_no DESC LIMIT 5
) AS backwards ORDER by backwards.emp_no ASC;
```

Note that that querying the first page has no cursors and just does a limit query (e.g. if the offset was 0) while subsequent queries have a WHERE constraint using the cursor.

```log
MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 20; 
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10001 | Georgi     | Facello     | M      |
|  10003 | Parto      | Bamford     | M      |
|  10004 | Chirstian  | Koblick     | M      |
|  10005 | Kyoichi    | Maliniak    | M      |
|  10008 | Saniya     | Kalloufi    | M      |
|  10012 | Patricio   | Bridgland   | M      |
|  10013 | Eberhardt  | Terkki      | M      |
|  10014 | Berni      | Genin       | M      |
|  10015 | Guoxiang   | Nooteboom   | M      |
|  10016 | Kazuhito   | Cappelletti | M      |
|  10019 | Lillian    | Haddadi     | M      |
|  10020 | Mayuko     | Warwick     | M      |
|  10021 | Ramzi      | Erde        | M      |
|  10022 | Shahaf     | Famili      | M      |
|  10025 | Prasadram  | Heyers      | M      |
|  10026 | Yongqiao   | Berztiss    | M      |
|  10028 | Domenick   | Tempesti    | M      |
|  10029 | Otmar      | Herbst      | M      |
|  10030 | Elvis      | Demeyer     | M      |
|  10031 | Karsten    | Joslin      | M      |
+--------+------------+-------------+--------+
20 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 5; 
+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10001 | Georgi     | Facello   | M      |
|  10003 | Parto      | Bamford   | M      |
|  10004 | Chirstian  | Koblick   | M      |
|  10005 | Kyoichi    | Maliniak  | M      |
|  10008 | Saniya     | Kalloufi  | M      |
+--------+------------+-----------+--------+
5 rows in set (0.002 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no > 10008 ORDER BY emp_no DESC LIMIT 5;
+--------+------------+--------------+--------+
| emp_no | first_name | last_name    | gender |
+--------+------------+--------------+--------+
| 499999 | Sachin     | Tsukuda      | M      |
| 499998 | Patricia   | Breugel      | M      |
| 499997 | Berhard    | Lenart       | M      |
| 499996 | Zito       | Baaz         | M      |
| 499993 | DeForest   | Mullainathan | M      |
+--------+------------+--------------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no > 10016 ORDER BY emp_no ASC LIMIT 5;
 (cursor_previous=10026) 
SELECT emp_no, first_nam+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10019 | Lillian    | Haddadi   | M      |
|  10020 | Mayuko     | Warwick   | M      |
|  10021 | Ramzi      | Erde      | M      |
|  10022 | Shahaf     | Famili    | M      |
|  10025 | Prasadram  | Heyers    | M      |
+--------+------------+-----------+--------+
5 rows in set (0.002 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no > 10025 ORDER BY emp_no ASC LIMIT 5;
+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10026 | Yongqiao   | Berztiss  | M      |
|  10028 | Domenick   | Tempesti  | M      |
|  10029 | Otmar      | Herbst    | M      |
|  10030 | Elvis      | Demeyer   | M      |
|  10031 | Karsten    | Joslin    | M      |
+--------+------------+-----------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM (
    ->     SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no < 10026 ORDER BY emp_no DESC LIMIT 5
    -> ) AS backwards ORDER by backwards.emp_no ASC; 
+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10019 | Lillian    | Haddadi   | M      |
|  10020 | Mayuko     | Warwick   | M      |
|  10021 | Ramzi      | Erde      | M      |
|  10022 | Shahaf     | Famili    | M      |
|  10025 | Prasadram  | Heyers    | M      |
+--------+------------+-----------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM (
    ->     SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no < 10019 ORDER BY emp_no DESC LIMIT 5
    -> ) AS backwards ORDER by backwards.emp_no ASC; 
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10012 | Patricio   | Bridgland   | M      |
|  10013 | Eberhardt  | Terkki      | M      |
|  10014 | Berni      | Genin       | M      |
|  10015 | Guoxiang   | Nooteboom   | M      |
|  10016 | Kazuhito   | Cappelletti | M      |
+--------+------------+-------------+--------+
5 rows in set (0.001 sec)

MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM (
    ->     SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' AND emp_no < 10012 ORDER BY emp_no DESC LIMIT 5
    -> ) AS backwards ORDER by backwards.emp_no ASC;
+--------+------------+-----------+--------+
| emp_no | first_name | last_name | gender |
+--------+------------+-----------+--------+
|  10001 | Georgi     | Facello   | M      |
|  10003 | Parto      | Bamford   | M      |
|  10004 | Chirstian  | Koblick   | M      |
|  10005 | Kyoichi    | Maliniak  | M      |
|  10008 | Saniya     | Kalloufi  | M      |
+--------+------------+-----------+--------+
5 rows in set (0.002 sec)
```

As you can see, at a high-level, the results of the queries are identical to those above, use of the cursor is (in this case) analogous to using the offset as earlier. In subsequent sections, we'll see the implications below. Unfortunately, the token/cursor pagination method has the same data consistency problem. It will present slightly different but pretty much be the same.

To summarize, token/cursor-based pagination is consistent, the query should take similar amounts of times for each page but will only allow sequential movement through the pages (1 -> 2 -> 3 -> 2). Alternatively, you could "calculate" all of the cursors on the first run and use that client-side to be able to randomly access the pages. The cursor "logic" isn't complicated, it's just the employee number of the first and last employee in the page.

### Using Actual Cursors

Believe it or not, most SQL databases have built-in implementations of cursors which do exactly what we're doing. Cursors [as a rule] will most likely be more performant and more functional, but they don't play well with the current push we have for abstracting database functionality from a given application. Cursors have a lot of gotchas/restrictions which can make implementation a pain and they may also require database side code to get any kind of functionality.

In general, when the kind of paging we're discussing is done, it's done on a request/response nature (asynchronous); many of the cursor implementations are meant to be synchronous/interactive/conversational which requires a fair bit of state management on the side of the application since things like transactions can't be maintained by a client unless it’s talking directly to the database. I think this more easily communicates that you must REALLY consider your use cases and if cursors can provide you the actual functionality you want that's practically _un-achievable_ with doing straight queries.

I think this requires a second mention: cursors have a lot of restrictions; there are different kinds of cursors and there are some things that the database just won't let you do with cursors. For example:

- [MariaDB](https://mariadb.com/kb/en/cursor-overview/): these cursors must be inside stored programs, are non-scrollable, read-only and asensitive
- [MySQL](https://dev.mysql.com/doc/refman/8.0/en/cursors.html): these queries must be done inside stored programs, they're read-only, asensitive and non-scrollable.
- [Postgres](https://www.postgresql.org/docs/current/plpgsql-cursors.html): cursors must be declared within a transaction, but scrollable cursors are supported along with write-able, but are insensitive

## Data Consistency

In the previous two sections, I've mentioned data consistency and how changes within the data set between reading pages can have unintended consequences when reading pages of data. This is a problem that's heavily influenced by the lifetime of your data, questions like:

 1. How often does my data set change?
 2. What's the purpose of reading the page?
 3. Am I trying to list it to mutate one of the items?
 4. am I listing it to create a data set as an input for something else?

The lifetime of your data and the set itself will determine how changes in data consistency will present when querying pages; changes in consistency can have a direct _effect_ on the data set itself but can additionally have an effect on what you want to do with the data page after you get it. For the purposes of this article data consistency can be summarized as:

- On read, how does the data changing affect the results?
- On write (e.g., after reading data), how can I be sure that I’m writing the data that i've read?

Depending on your answers to the question above and how data consistency affects your result (or what you use the results for); you can do SOME things to mitigate that affect. I think that most APIs ignore the results of inconsistent data on read because in general, that problem can be fixed by reading the data again; in addition, determining if the data has changed while reading is difficult/expensive (with straight queries) since you must read the entire data set somehow. [SQL cursors](#using-actual-cursors) handle this problem gracefully by having sensitive cursors (not supported by all databases).

The easiest way to know if the data set being read (as a whole, not just the page) has changed is to read the entire data set and compare it to the last time you've read the data set; I provide an example [below](#validating-data-consistency-on-read). On write, data consistency can be verified by adding a WHERE constraint using an audit field that's updated atomically (e.g. a version field or an updated_at field). Those kinds of fields are expected to be updated for each mutation and as a result if it's been mutated between your last read and write, your constraint will fail and there will be no rows modified.

### Validating Data Consistency on Read

One way to validate the consistency of the data set on read is to query values that shouldn't change during the lifetime of the query and hash them; if the hash changes between queries, it means your data set has changed and you should "refresh" the query. Below I’ll describe some of the use cases which should make it CLEAR why using total has some downsides:

- if an insert occurs that falls within your data set, a new "row" exists and the hash changes (the total would also change in this case)
- if a row is deleted that falls within your data set, an existing "row" is removed and the hash changes (the total would also change in this case)
- if a row is deleted, and then a row is inserted within the data set the hash changes (the total would remain the same in this case)

We can generate a hash/checksum with the employees table by doing the following:

```sql
SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
SET @query_md5 = (SELECT md5(@query_emp_nos));
SELECT @query_emp_nos as employee_numbers, @query_md5 as query_md5;
```

The above queries should have the following output:

```log
MariaDB [employees]> SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
Query OK, 0 rows affected (0.000 sec)

MariaDB [employees]> SET @query_md5 = (SELECT md5(@query_emp_nos));
Query OK, 0 rows affected (0.000 sec)

MariaDB [employees]> SELECT @query_emp_nos as employee_numbers, @query_md5 as query_md5;
+-------------------------------------------------------------------------------------------------------------------------+----------------------------------+
| employee_numbers                                                                                                        | query_md5                        |
+-------------------------------------------------------------------------------------------------------------------------+----------------------------------+
| 10001,10002,10003,10004,10005,10006,10007,10008,10009,10010,10011,10012,10013,10014,10015,10016,10017,10018,10019,10020 | 042d7815a601d13aabd0bd93da566133 |
+-------------------------------------------------------------------------------------------------------------------------+----------------------------------+
1 row in set (0.001 sec)
```

Recall the queries below from the sections above, we can re-create the checksum each time and tell when the checksum changes.

```sql
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 20;
SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
SET @query_md5 = (SELECT md5(@query_emp_nos));
SELECT @query_md5 as query_checksum_original;

SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 10 OFFSET 0;
SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
SET @query_md5 = (SELECT md5(@query_emp_nos));
SELECT @query_md5 as query_checksum_after_page_1;

INSERT INTO employees(emp_no,birth_date,first_name,last_name,gender,hire_date) values('10000', '2022-10-02', 'John', 'Smith', 'M', '2022-10-02');
SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
SET @query_md5 = (SELECT md5(@query_emp_nos));
SELECT @query_md5 as query_checksum_after_insert;

SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 10 OFFSET 10;
SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
SET @query_md5 = (SELECT md5(@query_emp_nos));
SELECT @query_md5 as query_checksum_after_page_2;

DELETE FROM employees WHERE emp_no='10000';
SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
SET @query_md5 = (SELECT md5(@query_emp_nos));
SELECT @query_md5 as query_checksum_after_delete;
```

```log
MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 20;
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10001 | Georgi     | Facello     | M      |
|  10003 | Parto      | Bamford     | M      |
|  10004 | Chirstian  | Koblick     | M      |
|  10005 | Kyoichi    | Maliniak    | M      |
|  10008 | Saniya     | Kalloufi    | M      |
|  10012 | Patricio   | Bridgland   | M      |
|  10013 | Eberhardt  | Terkki      | M      |
|  10014 | Berni      | Genin       | M      |
|  10015 | Guoxiang   | Nooteboom   | M      |
|  10016 | Kazuhito   | Cappelletti | M      |
|  10019 | Lillian    | Haddadi     | M      |
|  10020 | Mayuko     | Warwick     | M      |
|  10021 | Ramzi      | Erde        | M      |
|  10022 | Shahaf     | Famili      | M      |
|  10025 | Prasadram  | Heyers      | M      |
|  10026 | Yongqiao   | Berztiss    | M      |
|  10028 | Domenick   | Tempesti    | M      |
|  10029 | Otmar      | Herbst      | M      |
|  10030 | Elvis      | Demeyer     | M      |
|  10031 | Karsten    | Joslin      | M      |
+--------+------------+-------------+--------+
20 rows in set (0.000 sec)

MariaDB [employees]> SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
Query OK, 0 rows affected (0.000 sec)

MariaDB [employees]> SET @query_md5 = (SELECT md5(@query_emp_nos));
Query OK, 0 rows affected (0.000 sec)

MariaDB [employees]> SELECT @query_md5 as query_checksum_original;
+----------------------------------+
| query_checksum_original          |
+----------------------------------+
| 042d7815a601d13aabd0bd93da566133 |
+----------------------------------+
1 row in set (0.000 sec)

MariaDB [employees]> 
MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 10 OFFSET 0;
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10001 | Georgi     | Facello     | M      |
|  10003 | Parto      | Bamford     | M      |
|  10004 | Chirstian  | Koblick     | M      |
|  10005 | Kyoichi    | Maliniak    | M      |
|  10008 | Saniya     | Kalloufi    | M      |
|  10012 | Patricio   | Bridgland   | M      |
|  10013 | Eberhardt  | Terkki      | M      |
|  10014 | Berni      | Genin       | M      |
|  10015 | Guoxiang   | Nooteboom   | M      |
|  10016 | Kazuhito   | Cappelletti | M      |
+--------+------------+-------------+--------+
10 rows in set (0.001 sec)

MariaDB [employees]> SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
Query OK, 0 rows affected (0.000 sec)

MariaDB [employees]> SET @query_md5 = (SELECT md5(@query_emp_nos));
Query OK, 0 rows affected (0.000 sec)

MariaDB [employees]> SELECT @query_md5 as query_checksum_after_page_1;
+----------------------------------+
| query_checksum_after_page_1      |
+----------------------------------+
| 042d7815a601d13aabd0bd93da566133 |
+----------------------------------+
1 row in set (0.000 sec)

MariaDB [employees]> 
MariaDB [employees]> INSERT INTO employees(emp_no,birth_date,first_name,last_name,gender,hire_date) values('10000', '2022-10-02', 'John', 'Smith', 'M', '2022-10-02');
ECT group_concat(emp_no) FROM (SELECT emp_no FROM Query OK, 1 row affected (0.004 sec)

MariaDB [employees]> SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
Query OK, 0 rows affected (0.001 sec)

MariaDB [employees]> SET @query_md5 = (SELECT md5(@query_emp_nos));
Query OK, 0 rows affected (0.001 sec)

MariaDB [employees]> SELECT @query_md5 as query_checksum_after_insert;
+----------------------------------+
| query_checksum_after_insert      |
+----------------------------------+
| 9b71c0a13c38afa1a6d6b932ff93c82a |
+----------------------------------+
1 row in set (0.001 sec)

MariaDB [employees]> 
MariaDB [employees]> SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender='M' ORDER BY emp_no ASC LIMIT 10 OFFSET 10;
+--------+------------+-------------+--------+
| emp_no | first_name | last_name   | gender |
+--------+------------+-------------+--------+
|  10016 | Kazuhito   | Cappelletti | M      |
|  10019 | Lillian    | Haddadi     | M      |
|  10020 | Mayuko     | Warwick     | M      |
|  10021 | Ramzi      | Erde        | M      |
|  10022 | Shahaf     | Famili      | M      |
|  10025 | Prasadram  | Heyers      | M      |
|  10026 | Yongqiao   | Berztiss    | M      |
|  10028 | Domenick   | Tempesti    | M      |
|  10029 | Otmar      | Herbst      | M      |
|  10030 | Elvis      | Demeyer     | M      |
+--------+------------+-------------+--------+
10 rows in set (0.000 sec)

MariaDB [employees]> SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
Query OK, 0 rows affected (0.000 sec)

MariaDB [employees]> SET @query_md5 = (SELECT md5(@query_emp_nos));
Query OK, 0 rows affected (0.000 sec)

MariaDB [employees]> SELECT @query_md5 as query_checksum_after_page_2;
+----------------------------------+
| query_checksum_after_page_2      |
+----------------------------------+
| 9b71c0a13c38afa1a6d6b932ff93c82a |
+----------------------------------+
1 row in set (0.000 sec)

MariaDB [employees]> 
MariaDB [employees]> DELETE FROM employees WHERE emp_no='10000';
Query OK, 1 row affected (0.003 sec)

MariaDB [employees]> SET @query_emp_nos = (SELECT group_concat(emp_no) FROM (SELECT emp_no FROM employees LIMIT 20) AS emp_no);
Query OK, 0 rows affected (0.001 sec)

MariaDB [employees]> SET @query_md5 = (SELECT md5(@query_emp_nos));
Query OK, 0 rows affected (0.001 sec)

MariaDB [employees]> SELECT @query_md5 as query_checksum_after_delete;
+----------------------------------+
| query_checksum_after_delete      |
+----------------------------------+
| 042d7815a601d13aabd0bd93da566133 |
+----------------------------------+
1 row in set (0.001 sec)
```

You'll notice that the query_checksum changes twice, (1) when the row is inserted, the checksum changes and (2) when the row is deleted the checksum changes back. This is a 100% solution to correct the problem if it's efficient to query a subset of the entire query whenever you want to know if the result of the query has changed. I think this is best implemented ad hoc; you should create a TTL (time to live) for a given paginated query such that you verify the checksum if the TTL has expired rather than each time you switch pages. This certainly has the downside that your query could be "invalid" before that TTL expires but querying the entire data set (even if it's only a single column) could be intensive at scale.

## Performance Considerations

Performance is a difficult topic; there are tons of variables that determine whether a query executes quickly or slowly and in the words of my [troglodyte](https://www.youtube.com/watch?v=2AzAFqrxfeY) DBA friend: _it depends_. Although I’ll make a very big generalization, it's safe to say (I’ll demo below) that token/cursor pagination is generally more efficient than offset based pagination because token/cursor-based pagination is less likely to have to scan the entire table to find the rows you're interested in while offset based pagination generally has to scan the entire table no matter what.

Before we get into the specifics, I think it's also important to note that the only index that exists on this table is the primary key (emp_no). This has a big effect on the overall optimization of the queries (in that there is little to no optimization):

```sql
SHOW INDEXES FROM employees;
```

```log
MariaDB [employees]> SHOW INDEXES FROM employees;
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+
| Table     | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Ignored |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+
| employees |          0 | PRIMARY  |            1 | emp_no      | A         |      299202 |     NULL | NULL   |      | BTREE      |         |               | NO      |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+
1 row in set (0.001 sec)
```

I'm going to provide a handful of queries and attempt to do some minor compare/contrast on what makes them bad/good; in this case we're not even worried about indexes, only that we use what we currently have. To start, I'm going to use a query that's really really bad, you should never use this query:

```sql
EXPLAIN SELECT * FROM employees LIMIT 10;
EXPLAIN SELECT * FROM employees;
```

```log
MariaDB [employees]> EXPLAIN SELECT * FROM employees LIMIT 10;
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 292202 |       |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
1 row in set (0.001 sec)

MariaDB [employees]> EXPLAIN SELECT * FROM employees;
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 292202 |       |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
1 row in set (0.001 sec)
```

While quick, this query can be considered bad because the type is "ALL"; this means that it had to scan the entire table (keep in mind that there are some cases where scanning the entire table is faster than other options). You can see that although we did two very different queries, the explain has the same results. Also note that the possible_keys field is null (meaning that it couldn't use an existing index).

The below query attempts to approximate offset/limit-based pagination (we're just querying employee number to nudge our explain in the right direction)

```sql
EXPLAIN SELECT emp_no FROM employees LIMIT 10;
EXPLAIN SELECT emp_no FROM employees LIMIT 10 OFFSET 10;
```

```log
MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees LIMIT 10;
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | index | NULL          | PRIMARY | 4       | NULL | 292202 | Using index |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
1 row in set (0.000 sec)

MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees LIMIT 10 OFFSET 10;
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | index | NULL          | PRIMARY | 4       | NULL | 292202 | Using index |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
1 row in set (0.000 sec)
```

With these queries it’s a bit better, we see that it used the primary key and although it had to go through a lot of rows, the type is index (instead of all). This is a better query and should generally be more performant.

Next, let’s do an explain on the cursor/pagination queries, we'll see something slightly different:

```sql
EXPLAIN SELECT emp_no FROM employees WHERE emp_no > 10010 LIMIT 10; 
EXPLAIN SELECT emp_no FROM employees WHERE emp_no < 10010 LIMIT 10; 
```

```log
MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees WHERE emp_no > 10010 LIMIT 10; 
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+--------------------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows   | Extra                    |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+--------------------------+
|    1 | SIMPLE      | employees | range | PRIMARY       | PRIMARY | 4       | NULL | 146101 | Using where; Using index |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+--------------------------+
1 row in set (0.001 sec)

MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees WHERE emp_no < 10010 LIMIT 10; 
+------+-------------+-----------+-------+---------------+---------+---------+------+------+--------------------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows | Extra                    |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+--------------------------+
|    1 | SIMPLE      | employees | range | PRIMARY       | PRIMARY | 4       | NULL | 9    | Using where; Using index |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+--------------------------+
1 row in set (0.001 sec)
```

Looking at these logs, you can see that there's a __BIG__ difference in the number of rows scanned for both queries, although both are able to use the primary key, the second query has to scan far fewer rows. We can imagine that the second query is the most performant out of the two. You'll also find that to be able to maintain queries like this you'll need to pre-calculate all the cursors (which has its own problems).

The "most" performant query would be this:

```sql
-- page 1->2 (cursor_next=10016, cursor_previous=10012)
EXPLAIN SELECT emp_no FROM employees WHERE emp_no BETWEEN 10012 AND 10016 ORDER BY emp_no ASC;
EXPLAIN SELECT emp_no, first_name, last_name, gender FROM employees WHERE emp_no BETWEEN 10012 AND 10016 ORDER BY emp_no ASC;
```

```log
MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees WHERE emp_no BETWEEN 10012 AND 10016 ORDER BY emp_no ASC;
+------+-------------+-----------+-------+---------------+---------+---------+------+------+--------------------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows | Extra                    |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+--------------------------+
|    1 | SIMPLE      | employees | range | PRIMARY       | PRIMARY | 4       | NULL | 5    | Using where; Using index |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+--------------------------+
1 row in set (0.001 sec)

MariaDB [employees]> EXPLAIN SELECT emp_no, first_name, last_name, gender FROM employees WHERE emp_no BETWEEN 10012 AND 10016 ORDER BY emp_no ASC;
+------+-------------+-----------+-------+---------------+---------+---------+------+------+-------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+-------------+
|    1 | SIMPLE      | employees | range | PRIMARY       | PRIMARY | 4       | NULL | 5    | Using where |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+-------------+
1 row in set (0.000 sec)
```

We can assume that this is the most performant query because if we define two fixed ranges, using the indexes, we can limit the number of rows that need to be scanned to what rows are between the constraints.
