---
title: "[fluetd] S3にアップロードされたキー名でルーティングする"
date: 2021-10-26
draft: false
categories:
- ETL
---

S3にアップロードされたファイルをfluentdでBigQueryにinsertする際、S3キー名に応じてテーブルを振り分けるサンプルを掲載します。
ここではフォーマットは`s3://my-bucket/{BigQueryデータセット名}/{テーブル名}/{uuid}.csv.gz` とします。

## fluent.conf

```xml:fluent.conf
<source>
  tag s3
  @type s3
  s3_bucket my-bucket
  s3_region ap-northeast-1
  <sqs>
    queue_name my-queue
  </sqs>
</source>
<match s3>
  @type rewrite_tag_filter
  <rule>
    key s3_key
    pattern ^(.+?)/(.+?)/.+\.gz$
    tag bigquery.$1.$2
  </rule>
</match>
<filter bigquery.hoge.fuga>
    @type parser
    key_name message
    <parse>
        @type csv
        keys id,family_name,first_name
        null_empty_string true
    </parse>
</filter>
<filter bigquery.foo.bar>
    @type parser
    key_name message
    <parse>
        @type csv
        keys column_1,column_2
        null_empty_string true
    </parse>
</filter>
<match bigquery.*>
  @type bigquery_insert
  project my-project
  dataset ${tag[1]}
  table ${tag[2]}
  auth_method application_default
  fetch_schema true
  <buffer tag>
    @type memory
  </buffer>
</match>
```

## 解説

### source

```xml:fluent.conf
<source>
  tag s3
  @type s3
  s3_bucket my-bucket
  s3_region ap-northeast-1
  add_object_metadata true
  <sqs>
    queue_name my-queue
  </sqs>
</source>
```

以下のファイルをアップロードしたとします。

```csv:s3//my-bucket/hoge/fuga/uuid.csv.gz
1, tanaka, taro
```

`add_object_metadata` オプションにより、以下の形にtransformされます。

```:output
tag:s3 {"message":"1,tanaka,taro", "s3_bucket":"my-bucket", "s3_key":"hoge/fuga/uuid.csv.gz"}
```

### rewrite_tag_filter

```xml:fluent.conf
<match s3>
  @type rewrite_tag_filter
  <rule>
    key s3_key
    pattern ^(.+?)/(.+?)/.+\.gz$
    tag bigquery.$1.$2
  </rule>
</match>
```

s3_keyを正規表現で展開し、タグが書き換えられます。

```:output
tag:bigquery.hoge.fuga {"message":"1,tanaka,taro", "s3_bucket":"my-bucket", "s3_key":"hoge/fuga/uuid.csv.gz"}
```

### filter

```xml:fluent.conf
<filter bigquery.hoge.fuga>
    @type parser
    key_name message
    <parse>
        @type csv
        keys id,family_name,first_name
        null_empty_string true
    </parse>
</filter>
```

message以外のメタデータを捨て、csvデータをパースします。

```
tag:bigquery.hoge.fuga {"id":"1", "family_name":"tanaka", "first_name":"taro"}
```

### bigquery_insert

```xml:fluent.conf
<match bigquery.*>
  @type bigquery_insert
  project my-project
  dataset ${tag[1]}
  table ${tag[2]}
  auth_method application_default
  fetch_schema true
  <buffer tag>
    @type memory
  </buffer>
</match>
```

タグからdatasetとtableを取得し、対象のテーブルにレコードをstreaming insertします。
tagプレースホルダを使用するためには `<buffer tag>` の宣言が必要になります。

`my-project:hoge.fuga`

|id|family_name|first_name|
|-|-|-|
|1|tanaka|taro|
