---
title: "BigQuery日付別テーブルをまとめてDROPするSQL"
date: 2025-02-14
draft: false
categories:
- BigQuery
- 備忘録
---

```sql
FOR record IN (SELECT DISTINCT _TABLE_SUFFIX AS table_suffix FROM `dataset.table_*`)
DO
  EXECUTE IMMEDIATE "DROP TABLE `dataset.table_" || record.table_suffix || "`";
END FOR;
```
