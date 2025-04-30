---
title: "max_stalenessを一括変更するSQL"
date: 2025-04-30
draft: false
categories:
- BigQuery
- 備忘録
---

Datastreamは `max_staleness` オプションを指定することでDatastreamが作成するテーブルにその値が反映されます。
https://cloud.google.com/datastream/docs/destination-bigquery?hl=ja#use-max-staleness

しかし、Datastreamのmax_stalenessの値を後から変更しても既に作成されたテーブルには反映されません。
以下のようなSQLで手動で変更する必要があります。

```sql
FOR tables IN (SELECT DISTINCT table_schema || "." || table_name AS table_id FROM `region-asia-northeast1`.INFORMATION_SCHEMA.TABLE_OPTIONS WHERE option_name = 'max_staleness' AND table_schema = 'データセット名')
DO
  EXECUTE IMMEDIATE "ALTER TABLE " || tables.table_id || " SET OPTIONS (max_staleness = INTERVAL 1 DAY);";
END FOR;
```
