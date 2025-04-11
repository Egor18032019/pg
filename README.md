"# pg"

[1] ускорить простой запроc, добиться времени выполнения < 10ms
```shell
EXPLAIN ANALYZE
select name from t1 where id = 50000;
```
Первоначальное время выполнение составило: 113,221ms![1 report.png](../1%20report.png)
Проблема: Медленно
Решение: Был применен индекс
```shell
CREATE INDEX CONCURRENTLY t1_id_idx ON t1(id);
```
Итоговое время выполнения составило: 0,032ms![1.1 report.png](../1.1%20report.png)

[2] ускорить запрос "max + left join", добиться времени выполнения < 10ms
```shell
EXPLAIN ANALYZE
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```
Первоначальное время выполнение составило: 3138,215ms ![2.0 report.jpg](../2.0%20report.jpg)
Проблема: Очень медленно
Решение: Создать доп.индекс на t2(id) и t1(name)
```shell
CREATE INDEX CONCURRENTLY t1_id_idx ON t2(id);
```
```shell
CREATE INDEX t1_name_idx ON t1(name);
```
После индексов время выполнения составило  3065.928 ms   ![2.1 report.jpg](../2.1%20report.jpg)
- что не достаточно.
Далее добавим еще индексов
```shell
CREATE INDEX idx_t2_day ON t2(day);
```
Итог 3135.981 ms ![2.2 report.jpg](../2.2%20report.jpg)
- что не достаточно.
Далее добавим еще индексов
```shell
CREATE INDEX t2_covering_idx ON t2(t_id, day DESC);
```
Итог 3117.515 ms ![2.3 report.jpg](../2.3%20report.jpg)
 