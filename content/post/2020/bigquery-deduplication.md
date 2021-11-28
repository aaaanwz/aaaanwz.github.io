---
title: "BigQueryで重複レコードを削除するSQL"
date: 2020-02-18
draft: false
categories:
- BigQuery
- 小ネタ
---

オペミスやat-least-onceセマンティクスによってINSERTされてしまった重複レコードを消すSQLです。

# 完全に同一な重複レコードを消す

やる事は
1. 重複レコードのうち最古のものを一時テーブルに退避
2. 重複レコードを全て削除
3. 一時テーブルから再度INSERT
です。


`Schema`

```json
[
  {
    "mode": "REQUIRED",
    "name": "id",
    "type": "INTEGER"
  },
  {
    "mode": "NULLABLE",
    "name": "value",
    "type": "STRING"
  }
]
```


```sql
CREATE TABLE project_name.tmp OPTIONS(
    expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
) AS(
    SELECT
        id,
        value
    FROM
        (
            SELECT
                *,
                COUNT(id) OVER(PARTITION BY id) AS count,
                ROW_NUMBER() OVER(PARTITION BY id) AS row_number
            FROM
                project_name.table_name
        )
    WHERE
        count > 1
    AND row_number = 1
);

DELETE
FROM
    project_name.table_name
WHERE
    id IN(
        SELECT
            id
        FROM
            project_name.tmp
    );

INSERT INTO project_name.table_name(
    SELECT
        *
    FROM
        project_name.tmp
);

DROP TABLE project_name.tmp;
```

残すレコードが最古のものでなくても構わないのであれば、最初のCREATE は少し単純化できます。

```sql
CREATE TABLE project_name.tmp OPTIONS(
    expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
) AS(
    SELECT
        id,
        value
    FROM
        (
            SELECT
                *,
                ROW_NUMBER() OVER(PARTITION BY id) AS row_number
            FROM
                project_name.table
        )
    WHERE row_number = 2
);
```

---

# 一部が異なるレコードを消す

同一 `id`で `timestamp` が違うレコードが存在する場合、最も古いレコード以外を削除するクエリです。

```sql
MERGE dataset_name.table_name table
USING
    (
        SELECT
            id,
            MIN(timestamp) AS timestamp
        FROM
            dataset_name.table_name
        GROUP BY
            id
        HAVING COUNT(id) > 1
    ) duplicated_origin
ON  table.id = duplicated_origin.id
AND table.timestamp != duplicated_origin.timestamp
WHEN MATCHED THEN
    DELETE
;
```
