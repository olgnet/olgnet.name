---
title:  "How the ESR (Equality, Sort, Range) rule works for PostgreSQL"
date:   2023-11-15 23:48:51 +0300
---
[The official MongoDB documentation](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-rule/) recommends following the ESR (Equality, Sort, Range) rule when creating indexes. 

According to the ESR rule, the order of fields in the index should be as follows:
1. **[Equality]** Fields for which regular equality is used
2. **[Sort]** Fields for which sorting is applied
3. **[Range]** Fields that require a range search (between, in, etc.)

The rule is well remembered by developers and is relevant not only for MongoDB. Let's see how it works using PostgreSQL as an example.

## Test data

For experiments, we will use PostgreSQL 16.0 with a test table **task** and 10 million random records:

```sql
create table task(
	id     varchar   not null,
	status varchar   not null,
	date   timestamp not null,
	type   varchar   not null,
	price  bigint    not null
)
```

Let's try to optimize the following query:

```sql
select * 
from task 
where type = 'MEASURE' 
and price between 4500000 and 6000000 
order by date desc 
limit 100
```

## Attempt 1. Parallel Seq Scan

```sql
Limit  (cost=203712.99..203724.66 rows=100 width=68) (actual time=341.548..344.336 rows=100 loops=1)
  ->  Gather Merge  (cost=203712.99..232545.20 rows=247116 width=68) (actual time=338.486..341.267 rows=100 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Sort  (cost=202712.96..203021.86 rows=123558 width=68) (actual time=316.398..316.402 rows=78 loops=3)
              Sort Key: date DESC
              Sort Method: top-N heapsort  Memory: 47kB
              Worker 0:  Sort Method: top-N heapsort  Memory: 48kB
              Worker 1:  Sort Method: top-N heapsort  Memory: 47kB
              ->  Parallel Seq Scan on task  (cost=0.00..197990.67 rows=123558 width=68) (actual time=3.426..308.676 rows=100070 loops=3)
                    Filter: ((price >= 4500000) AND (price <= 6000000) AND ((type)::text = 'MEASURE'::text))
                    Rows Removed by Filter: 3233263
Planning Time: 0.116 ms
Execution Time: 344.716 ms
```

The query took more than 300 ms to complete, obviously we need an index.

## Attempt 2. Naive index

```sql
create index idx on task (type, price, date)
```

```sql
Limit  (cost=195097.54..195109.21 rows=100 width=68) (actual time=440.948..444.467 rows=100 loops=1)
  ->  Gather Merge  (cost=195097.54..223929.75 rows=247116 width=68) (actual time=437.915..441.429 rows=100 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Sort  (cost=194097.52..194406.41 rows=123558 width=68) (actual time=418.859..418.865 rows=78 loops=3)
              Sort Key: date DESC
              Sort Method: top-N heapsort  Memory: 47kB
              Worker 0:  Sort Method: top-N heapsort  Memory: 47kB
              Worker 1:  Sort Method: top-N heapsort  Memory: 46kB
              ->  Parallel Bitmap Heap Scan on task  (cost=9921.42..189375.22 rows=123558 width=68) (actual time=60.714..411.669 rows=100070 loops=3)
                    Recheck Cond: (((type)::text = 'MEASURE'::text) AND (price >= 4500000) AND (price <= 6000000))
                    Rows Removed by Index Recheck: 1700508
                    Heap Blocks: exact=16587 lossy=23144
                    ->  Bitmap Index Scan on idx  (cost=0.00..9847.28 rows=296538 width=0) (actual time=68.262..68.262 rows=300210 loops=1)
                          Index Cond: (((type)::text = 'MEASURE'::text) AND (price >= 4500000) AND (price <= 6000000))
Planning Time: 0.126 ms
Execution Time: 444.798 ms
```

The query plan can be described as follows:
1. **[Bitmap Index Scan]** Performs a search using the index prefix **(type, price)**. Since the row bitmap does not fit in memory (work_mem), a page bitmap is built. It marks pages that contain rows that match the condition
2. **[Parallel Bitmap Heap Scan]** Parallel filtering of rows is performed according to the bitmap
3. **[Sort]** Each resulting set of rows is sorted by **date**
4. **[Gather Merge]** Merges parallel sorted sets of rows
5. **[Limit]** The first 100 rows must be extracted from the resulting set

Thus, the main reason for the query slowdown was the blocking sort on the **date** field.
## Attempt 3. ESR rule index

```sql
create index idx on task (type, date, price)
```

```sql
Limit  (cost=0.56..189.03 rows=100 width=68) (actual time=0.062..0.471 rows=100 loops=1)
  ->  Index Scan Backward using idx on task  (cost=0.56..558882.80 rows=296538 width=68) (actual time=0.061..0.461 rows=100 loops=1)
        Index Cond: (((type)::text = 'MEASURE'::text) AND (price >= 4500000) AND (price <= 6000000))
Planning Time: 0.417 ms
Execution Time: 0.520 ms
```

Done, less than 1 ms to complete the request. In this case, we did not need to perform blocking sort and build a bitmap.

## Expression `where column in (…)`

```sql
select * 
from task 
where type = 'MEASURE' 
and status in ('CREATED', 'CLOSED', 'EXECUTED') 
order by date desc 
limit 100
```

When optimizing this query, the expression on the **status** field should be treated as Range (R), 
since it requires multiple passes through the index. Thus, the optimal index for optimizing such a query would be:
```sql
create index idx on task (type, date, status)
```

## Conclusion

The ESR rule is relevant for any B-Tree indexes and provides a convenient mnemonics for developers. 
It is not applicable in all cases, but can be a basis for further index optimization.