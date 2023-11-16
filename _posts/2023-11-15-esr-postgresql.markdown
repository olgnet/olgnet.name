---
layout: post
title:  "ESR правило для PostgreSQL"
date:   2023-11-15 23:48:51 +0300
---
В официальной документации MongoDB при создании индексов рекомендуется следовать правилу ESR (Equality, Sort, Range). Суть правила в правильном порядке полей в индексе:

1. [Equality] Поля, для которых в запросе используется обычное равенство
2. [Sort] Поля, для которых применяется сортировка
3. [Range] Поля, для которых требуется поиск по диапазону (between, in и т.д.)

На самом деле ESR правило актуально и для других баз данных, например, для PostgreSQL.

## Тестовые данные

Для экспериментов будем использовать PostgreSQL 16.0 с тестовой табличкой **task** и 10 миллионами случайных записей:

```sql
create table task(
	id     varchar   not null,
	status varchar   not null,
	date   timestamp not null,
	type   varchar   not null,
	price  bigint    not null
)
```

Попробуем оптимизировать следующий запрос:

```sql
select * 
from task 
where type = 'MEASURE' 
and price between 4500000 and 6000000 
order by date desc 
limit 100
```

## Попытка 1. Parallel Seq Scan

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

Без индексов выполнение запроса заняло больше 300 мс, попробуем создать индекс.

## Попытка 2. Наивный индекс

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

Полученный план запроса можно описать следующий образом:

1) **[Bitmap Index Scan]** Выполняется поиск по префиксу индекса **(type, price)**. ****Так как построчная битовая карта не помещается в памяти (work_mem), выполняется построение постраничной битовой карты. В ней отмечены страницы, в которых есть строки, соответствующие условию

2) **[Parallel Bitmap Heap Scan]** Выполняется параллельная фильтрация строк, согласно построенной на основе индекса битовой карте (Parallel Bitmap Heap Scan).

3) **[Sort]** Каждый полученный набор строк сортируется по **date**.

4) **[Gather Merge]** Выполняется слияние параллельно отсортированных наборов строк.

5) **[Limit]** Из полученного набора необходимо извлечь первые 100 строк.

Таким образом, основной причиной замедления запроса стала блокирующая сортировка по полю **date**.

## Попытка 3. Индекс по ESR правилу

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

Выполнение запроса заняло меньше 1 мс. В плане мы видим только Index Scan Backward и Limit.

## Выражение where column in (…)

```sql
select * 
from task 
where type = 'MEASURE' 
and status in ('CREATED', 'CLOSED', 'EXECUTED') 
order by date desc 
limit 100
```

При оптимизации этого запроса выражение по полю **status** следует рассматривать как Range (R), так как оно требует несколько проходов по индексу. Таким образом, оптимальный индексом для оптимизации подобного запроса будет:

```sql
create index idx on task (type, date, status)
```

## Заключение

ESR правило актуально для любого B-Tree индекса, в первую очередь из-за дороговизны блокирующей сортировки.
