```shell
DROP INDEX idx_t1_id;
DROP INDEX t1_name_idx;
DROP INDEX t2_covering_idx;
DROP INDEX t1_id_idx;
DROP FUNCTION IF EXISTS random(bigint, bigint);
```
```shell
\df
```
```shell
\dti
```
EXPLAIN ANALYZE
SELECT MAX(day)
FROM (
SELECT t2.day
FROM t2
JOIN t1 ON t2.t_id = t1.id
WHERE t1.name LIKE 'a%'
) AS subquery;
   
