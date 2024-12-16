---
title: "Cloud ComposerのConnectionをエクスポートする方法"
date: 2024-12-16
draft: false
categories:
- データ基盤
- 小ネタ
---

## TL;DR

Cloud Composerで

```sh
$ airflow connections export connections.json`
```

を実行したい場合、

```sh
$ gcloud composer environments run {ENVIRONMENT_NAME} --location={LOCATION} connections export -- /home/airflow/gcs/data/connections.json
$ gsutil cat gs://{BUCKET_NAME}/data/connections.json
```

とする。


---

Cloud Composerに対してAirflow CLIを実行したい場合は以下のようなコマンドで実行できる。

```sh
$ gcloud composer environments run {ENVIRONMENT_NAME} --location={LOCATION} [COMMAND] -- [SUBCOMMAND]
```
 
しかしこの方法はCloud ComposerのバックエンドであるGKEにPodを立ち上げてAirflow CLIを実行しているため、

```sh
$ airflow connections export connections.json
```

のように手元にファイルを保存するコマンドを使うことができない。（Podの中にファイルが生成され、終了とともに消えてしまう）

Cloud Composerは `/home/airflow/gcs` にGCSをマウントしているため、そこにファイルを出力するようにすればOK。

```sh
$ gcloud composer environments run {ENVIRONMENT_NAME} --location={LOCATION} connections export -- /home/airflow/gcs/data/connections.json
$ gsutil cat gs://{BUCKET_NAME}/data/connections.json
```
