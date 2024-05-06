---
title: "BigQueryにおけるGA4データの扱いづらさを解決する"
date: 2024-05-06
draft: false
categories:
- BigQuery
- GoogleAnalytics
- Dataform
---

## 背景

Google AnalyticsをBigQueryに連携してSQLで分析しようという時、`event_params` や `user_properties` が 

```
ARRAY<
    STRUCT<
        key STRING, 
        value STRUCT<
            string_value STRING, 
            int_value INT64, 
            float_value FLOAT64, 
            double_value FLOAT64
        >
    >
>
```

という大変扱い辛い型のため、以下のようにサブクエリを多用したSQLを度々書いていく必要があります。

```sql
SELECT
  TIMESTAMP_MICROS(event_timestamp) event_timestamp,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location') AS page_location,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_title') AS page_title,
  ...
FROM
  `analytics_123456789.events_*`
WHERE
  _TABLE_SUFFIX BETWEEN '20240101' AND '20240131'
```

このSQLの結果をデータマートにすれば解決かと思いきや、event_paramsにプロパティを追加する度にデータマートをメンテナンスする必要があり運用が非常に面倒です。

## ビューで分析クエリを簡単にする

手軽な方法で問題のカラムをkey-value JSON型に変換することを考えます。

```sql
# 型変換のためのUDF
CREATE FUNCTION dataset.parse_ga_struct(object ARRAY<STRUCT<key STRING, value STRUCT<string_value STRING, int_value INT64, float_value FLOAT64, double_value FLOAT64>>>)
RETURNS JSON
LANGUAGE js AS """
  var result = {};
  for (const param of object) {
    for (const type in param.value) {
      if (param.value[type] !== null) {
        result[param.key] = param.value[type];
        break;
      }
    }
  }
  return result;
""";
```

```sql
# ビュー
CREATE VIEW dataset.google_analytics AS
SELECT
    PARSE_DATE('%Y-%m-%d', _TABLE_SUFFIX) AS date,
    TIMESTAMP_MICROS(event_timestamp) AS event_timestamp,
    dataset.parse_ga_struct(event_params) AS event_params,
    dataset.parse_ga_struct(user_properties) AS user_properties,
    * EXCEPT(event_timestamp, event_params, user_properties)
FROM
    `analytics_123456789.events_*`;
```

このビューを用いることで分析クエリは以下のようになります。  
dateの範囲を指定することでプルーニングも効きます。

```sql
SELECT
  event_timestamp,
  JSON_VALUE(event_params.page_location) AS page_location,
  JSON_VALUE(event_params.page_title) AS page_title
  ...
FROM
  dataset.google_analytics
WHERE
  date BETWEEN '2024-01-01' AND '2024-01-31'
```

## データマートにしてクエリコストを削減する

上記のビューはスロットやスキャン量といったコスト面の効率が悪く、更にプルーニングのためにevent_timestampではなくdateを用いる必要があるためやや面倒です。  
そのまま物理テーブルにすればよいかと思いきや、**GA4のデータは `event_timestamp`, `user_pseudo_id`, `ga_session_id`, `event_bundle_sequence_id`, `event_name` を連結キーにしてもユニークにはなりません。** event_timestampはイベントが発生した実際の時刻ではなくGA4のマイクロバッチウィンドウの時刻であるためです。  
従ってテーブル更新処理の冪等性を担保するにはひと工夫が必要です。

更に、例えば`events_20200101`テーブルがBigQueryに作成されるのは `2020-01-02` とは限りません。3日近く遅延するケースも確認されています。従って日次更新の際に前日分のテーブルが作成されていることを前提とすることはできません。

上記の点を抑えてDataformで実装すると以下のようになります。


UDFの定義  
`parse_ga_struct.sqlx`
```sql
config {
    type: "operations",
    hasOutput: true
}

CREATE OR REPLACE FUNCTION ${self()}(object ARRAY<STRUCT<key STRING, value STRUCT<string_value STRING, int_value INT64, float_value FLOAT64, double_value FLOAT64>>>)
RETURNS JSON
LANGUAGE js AS """
  var result = {};
  for (const param of object) {
    for (const type in param.value) {
      if (param.value[type] !== null) {
        result[param.key] = param.value[type];
        break;
      }
    }
  }
  return result;
""";

```

GA4テーブルの定義  
`google_analytics_raw.sqlx`
```sql
config {
    type: "declaration",
    schema: "analytics_123456789",
    name: "events_*"
}
```

データマートの定義  
`google_analytics.sqlx`
```sql
config {
  type: "incremental",
  uniqueKey: ["event_id"],
  bigquery: {
    partitionBy: "DATE(event_timestamp)",
    requirePartitionFilter: true,
    updatePartitionFilter: "event_timestamp >= event_timestamp_checkpoint",
    clusterBy: ["event_name"]
  }
}

pre_operations {
  # 作成されたテーブルのリストを取得
  DECLARE table_suffix_list DEFAULT (
    SELECT
        ARRAY_AGG(DISTINCT REPLACE(table_name,'events_',''))
    FROM
        analytics_123456789.INFORMATION_SCHEMA.TABLES
    WHERE
        REGEXP_CONTAINS(table_name, 'events_[0-9]{8}')
        ${when(incremental(), "AND DATE(creation_time, 'Asia/Tokyo') = DATE_SUB(CURRENT_DATE('Asia/Tokyo'), INTERVAL 1 DAY)"}
    );
  ---
  # 影響を受けるターゲットテーブルのパーティションを取得
  DECLARE event_timestamp_checkpoint DEFAULT (
    SELECT
        MIN(TIMESTAMP_MICROS(event_timestamp))
    FROM
        ${ref("google_analytics_raw")}
    WHERE
        _TABLE_SUFFIX IN UNNEST(table_suffix_list)
  );
}

SELECT
    FARM_FINGERPRINT(TO_JSON_STRING(t)) AS event_id, # 行全体をハッシュ化してユニークキーにする
    TIMESTAMP_MICROS(event_timestamp) AS event_timestamp,
    ${ref("parse_ga_struct")}(event_params) AS event_params,
    ${ref("parse_ga_struct")}(user_properties) AS user_properties,
    * EXCEPT(event_timestamp, event_params, user_properties)
FROM
    ${ref("google_analytics_raw")} t
WHERE
    _TABLE_SUFFIX IN UNNEST(table_suffix_list)
QUALIFY
    ROW_NUMBER() OVER(PARTITION BY event_id) = 1 # event_idで重複排除
```

このデータマートを運用することで分析が非常に快適になります。

```sql
SELECT
  event_timestamp,
  JSON_VALUE(event_params.page_location) AS page_location,
  JSON_VALUE(event_params.page_title) AS page_title
  ...
FROM
  dataset.google_analytics
WHERE
  DATE(event_timestamp,'Asia/Tokyo') BETWEEN '2024-01-01' AND '2024-01-31'
```