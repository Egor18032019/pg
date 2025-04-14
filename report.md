EXPLAIN ANALYZE
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';

```shell
psql --username=postgres --dbname=mizhka
```

DROP INDEX IF EXISTS t1_id_name_idx ;
DROP INDEX IF EXISTS t1_name_idx;
DROP INDEX IF EXISTS t2_t_id_day_idx;

CREATE INDEX t1_id_name_idx ON t1(id) WHERE name LIKE 'a%';
EXPLAIN ANALYZE
SELECT MAX(t2.day)
FROM t2
WHERE EXISTS (
SELECT 1 FROM t1
WHERE t1.id = t2.t_id
AND t1.name LIKE 'a%'
);

 