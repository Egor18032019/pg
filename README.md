"# pg"

```shell
docker run --name pg -p 5432:5432 -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=mishka -d postgres:11-alpine
```
```shell
docker exec -it b06093992ff2 bash
```
```shell
psql --username=postgres --dbname=mishka
```
[1] ускорить простой запроc, добиться времени выполнения < 10ms
```shell
EXPLAIN ANALYZE
select name from t1 where id = 50000;
```
Первоначальное время выполнение составило: 113,221ms![1 report.png](1%20report.png)
Проблема: Медленно
Решение: Был применен индекс
```shell
CREATE INDEX CONCURRENTLY t1_id ON t1(id);
```
Итоговое время выполнения составило: 0,032ms![1.1 report.png](1.1%20report.png)
**Что удовлетворяет условиям задачи.**

[2] ускорить запрос "max + left join", добиться времени выполнения < 10ms
```shell
EXPLAIN ANALYZE
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```
Первоначальное время выполнение составило: 3138,215ms ![2.0 report.jpg](2.0%20report.jpg)
Проблема: Очень медленно
Решение: Создать доп.индекс на t2(id) и t1(name)
```shell
CREATE INDEX CONCURRENTLY t2_id ON t2(id);
```
```shell
CREATE INDEX CONCURRENTLY t1_name ON t1(name);
```
После индексов время выполнения составило  3065.928 ms   ![2.1 report.jpg](2.1%20report.jpg)
- что не достаточно.

Далее добавим еще индексов

CREATE INDEX t2_day ON t2(day);

Итог 3135.981 ms ![2.2 report.jpg](2.2%20report.jpg)
- что не достаточно.
- 
Проверил работу этого же на другом ноуте с ОС Linux время запроса составило:`![2 report on Linux.png](2%20report%20on%20Linux.png)
**Planning Time: 1.145 ms
Execution Time: 1714.852 ms** что почти в два раза меньше чем на ноуте с Windows но все равное не достаточно.

Исправить запрос ?; 
Рассмотрим возможность использования INNER JOIN вместо LEFT JOIN.
Это может улучшить производительность, так как исключает ненужные строки:
```shell
EXPLAIN ANALYZE
SELECT MAX(t2.day)
FROM t2
INNER JOIN t1 ON t2.t_id = t1.id
WHERE t1.name LIKE 'a%';
```



[3] ускорить запрос "anti-join", добиться времени выполнения < 10sec
```shell
EXPLAIN ANALYZE
select day from t2 where t_id not in ( select t1.id from t1 );
```
Решение: Используем NOT EXISTS вместо NOT IN и создаем индекс на t1.id. 
```shell
CREATE INDEX t1_id ON t1(id);
```

```shell
EXPLAIN (ANALYZE, VERBOSE) 
SELECT day 
FROM t2 
WHERE NOT EXISTS (SELECT 1 FROM t1 WHERE t1.id = t2.t_id);
```
Итоговый запрос: Execution Time: 2944.707 ms  ![3 report Linux.png](3%20report%20Linux.png)
**Что удовлетворяет условиям.**

[4] ускорить запрос "semi-join", добиться времени выполнения < 10sec
```shell
EXPLAIN ANALYZE
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
Итог:
**Planning Time: 0.142 ms
Execution Time: 5598.496 ms**
Делаем индексы:
```shell
CREATE INDEX t1_id ON t1(id);
CREATE INDEX t2_day ON t2(day);
```
Итоговое время 5669.157 ms ![4 report.png](4%20report.png)
А если поправить запрос то
```shell
EXPLAIN ANALYZE
SELECT t2.day 
FROM t2 
WHERE EXISTS (
    SELECT 1 FROM t1 
    WHERE t1.id = t2.t_id
)
AND t2.day > to_char(date_trunc('day', now() - interval '1 month'), 'yyyymmdd');
```
- время выполнения составит Execution Time: 5084.128 ms

[5] ускорить работу "savepoint + update", добиться постоянной во времени производительности (число транзакций в секунду)
 

```shell
psql -c 'select txid_current(); select pg_sleep(3600);' &
```

```shell
pgbench -p 5433 -rn -P1 -c10 -T3600 -M prepared -f generate_100_subtrans.sql 2>&1 > generate_100_subtrans_pgbench.log
```
Идей нет.