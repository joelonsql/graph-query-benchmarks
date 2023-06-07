# graph-query-benchmarks
Benchmark experiments comparing PostgreSQL against other specialized graph databases

Experiments run on a MacBook Pro 16-inch 2021, Apple M1 Max, 64GB RAM.

```sh
wget https://snap.stanford.edu/data/soc-pokec-relationships.txt.gz
gunzip soc-pokec-relationships.txt.gz
createdb graphdb
psql graphdb
```

```sql
SELECT version();
-- PostgreSQL 15.2 on aarch64-apple-darwin21.6.0, compiled by Apple clang version 14.0.0 (clang-1400.0.29.102), 64-bit
ALTER SYSTEM SET work_mem TO '1GB';
ALTER SYSTEM SET shared_buffers TO '16GB';
ALTER SYSTEM SET effective_cache_size TO '48GB';
```

```sh
pg_ctl restart # or Stop/Start via e.g. Postgres.app
```

```sql
CREATE TABLE friendships (user_1 INT, user_2 INT);
\COPY friendships FROM 'soc-pokec-relationships.txt';
-- COPY 30622564
-- Time: 6281.634 ms (00:06.282)
CREATE TABLE users AS
SELECT user_1 AS id FROM friendships
UNION
SELECT user_2 FROM friendships;
-- SELECT 1632803
-- Time: 11167.841 ms (00:11.168)
ALTER TABLE users ADD PRIMARY KEY (id);
-- Time: 485.167 ms
ALTER TABLE friendships ADD PRIMARY KEY (user_1, user_2);
-- Time: 9070.741 ms (00:09.071)
```

User 5867 is the user with most friends (8763 friends).
We will use this as a worse-case in the benchmark.


```sql
SELECT user_1, COUNT(*) FROM friendships GROUP BY 1 ORDER BY 2 DESC LIMIT 1;
```
```
 user_1 | count
--------+-------
   5867 |  8763
(1 row)

Time: 1294.654 ms (00:01.295)
```

We will also pick the 100th most connected user, since it's not wise to base
a benchmark on the most extreme case.

```sql
SELECT user_1, COUNT(*) FROM friendships GROUP BY 1 ORDER BY 2 DESC LIMIT 1 OFFSET 100;
```
```
 user_1 | count
--------+-------
  73665 |   802
(1 row)

Time: 1302.048 ms (00:01.302)
```

```sql
WITH RECURSIVE friends_of_friends AS
(
    SELECT 
        ARRAY[5867::INTEGER] AS current,
        0 AS depth
    UNION ALL
    SELECT
        new_current,
        friends_of_friends.depth + 1
    FROM
        friends_of_friends
    CROSS JOIN LATERAL (
        SELECT
            array_agg(DISTINCT friendships.user_2) AS new_current
        FROM
            friendships
        WHERE
            user_1 = ANY(friends_of_friends.current)
    ) q
    WHERE
        friends_of_friends.depth < 3
)
SELECT
    cardinality(current)
FROM
    friends_of_friends
WHERE
    depth = 3;
;
```
```
 cardinality
-------------
     1035293
(1 row)

Time: 1782.385 ms (00:01.782)
```

```sql
WITH RECURSIVE friends_of_friends AS
(
    SELECT 
        ARRAY[73665::INTEGER] AS current,
        0 AS depth
    UNION ALL
    SELECT
        new_current,
        friends_of_friends.depth + 1
    FROM
        friends_of_friends
    CROSS JOIN LATERAL (
        SELECT
            array_agg(DISTINCT friendships.user_2) AS new_current
        FROM
            friendships
        WHERE
            user_1 = ANY(friends_of_friends.current)
    ) q
    WHERE
        friends_of_friends.depth < 3
)
SELECT
    cardinality(current)
FROM
    friends_of_friends
WHERE
    depth = 3;
;
```
```
 cardinality
-------------
      462847
(1 row)

Time: 298.856 ms
```



```sql
WITH RECURSIVE friends_of_friends AS
(
    SELECT 
        ARRAY[5867::INTEGER] AS current,
        ARRAY[5867::INTEGER] AS found,
        0 AS depth
    UNION ALL
    SELECT
        new_current,
        found || q.new_current,
        friends_of_friends.depth + 1
    FROM
        friends_of_friends
    CROSS JOIN LATERAL (
        SELECT
            array_agg(DISTINCT friendships.user_2) AS new_current
        FROM
            friendships
        WHERE
            user_1 = ANY(friends_of_friends.current) AND
            NOT EXISTS (SELECT 1 FROM unnest(friends_of_friends.found) WHERE unnest = friendships.user_2)
    ) q
    WHERE
        friends_of_friends.depth < 3
)
SELECT
    cardinality(current)
FROM
    friends_of_friends
WHERE
    depth = 3;
;
```
```
 cardinality
-------------
      804944
(1 row)

Time: 2152.884 ms (00:02.153)
```

```sql
WITH RECURSIVE friends_of_friends AS
(
    SELECT 
        ARRAY[73665::INTEGER] AS current,
        ARRAY[73665::INTEGER] AS found,
        0 AS depth
    UNION ALL
    SELECT
        new_current,
        found || q.new_current,
        friends_of_friends.depth + 1
    FROM
        friends_of_friends
    CROSS JOIN LATERAL (
        SELECT
            array_agg(DISTINCT friendships.user_2) AS new_current
        FROM
            friendships
        WHERE
            user_1 = ANY(friends_of_friends.current) AND
            NOT EXISTS (SELECT 1 FROM unnest(friends_of_friends.found) WHERE unnest = friendships.user_2)
    ) q
    WHERE
        friends_of_friends.depth < 3
)
SELECT
    cardinality(current)
FROM
    friends_of_friends
WHERE
    depth = 3;
;
```
```
 cardinality
-------------
      440483
(1 row)

Time: 335.988 ms
```
