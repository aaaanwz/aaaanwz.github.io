---
title: "JSONLファイルからBigQueryスキーマJSONを生成する"
date: 2023-01-17
draft: false
categories:
- 小ネタ
---


`script.sh`

```sh
#!/bin/bash
set -eu
PROJECT_ID=myproject
DATASET_NAME=temp
TABLE_NAME=$(basename $1 | sed 's/\.[^\.]*$//')
bq load --autodetect --source_format=NEWLINE_DELIMITED_JSON ${PROJECT_ID}:${DATASET_NAME}.${TABLE_NAME} $1
bq show --schema --format=prettyjson ${PROJECT_ID}:${DATASET_NAME}.${TABLE_NAME} > ${TABLE_NAME}.json
bq rm -f -t ${PROJECT_ID}:${DATASET_NAME}.${TABLE_NAME}
```

```sh
$ ./script.sh /path/to/mydata.jsonl
```

以下のファイルが出力される

`./mydata.json`

```json
[
  {
    "mode": "NULLABLE",
    "name": "id",
    "type": "INTEGER"
  }
  ...
]
```

もっといい方法ありそう