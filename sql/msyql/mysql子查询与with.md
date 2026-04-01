# 子查询与 with
{docsify-updated}

## 子查询


## WITH
MySQL 8.0+ 才支持的 `WITH` 语句 指的是 `CTE（Common Table Expression，公共表表达式）`，用于把一段查询结果当成临时结果集使用，能让 SQL 更清晰、更易维护。


```
WITH cte_name AS (
    SELECT ...
)
SELECT ...
FROM cte_name;
```



## 问题
```
update user
set a = 10
where id in (
    select id from user where name like '%wbb'
);
```
会报错： `You can’t specify target table ‘user’ for update in FROM clause`

MySQL 不允许：UPDATE user 同时在子查询里 SELECT ... FROM user

因为 MySQL 无法保证更新顺序和数据一致性。

解决方案：
1. 使用派生表
```
UPDATE user
SET a = 10
WHERE id IN (
    SELECT t.id
    FROM (
        SELECT id
        FROM user
        WHERE name LIKE '%wbb'
    ) t
);
```

2. 使用 JOIN（性能最好，强烈推荐）
```
UPDATE user u
JOIN user u2 ON u.id = u2.id
SET u.a = 10
WHERE u2.name LIKE '%wbb';
```

3. 使用临时表（适合复杂逻辑）
```
CREATE TEMPORARY TABLE tmp_ids AS
SELECT id FROM user WHERE name LIKE '%wbb';

UPDATE user
SET a = 10
WHERE id IN (SELECT id FROM tmp_ids);
```