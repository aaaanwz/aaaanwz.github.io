---
title: "Apache Airflowのstandalone環境でHTTP接続が無期限にハングする問題"
date: 2023-12-22
draft: false
categories:
- 小ネタ
---

## 概要
Mac OS環境にて `airflow standalone` でAirflowを立ち上げると、DAGでHTTP接続を行おうとした際に無期限にハングする。

## 原因
OSXのPythonのバグ

## 回避策
Airflowを立ち上げる際に `NO_PROXY="*"` をセットする

```sh
$ export NO_PROXY="*"
$ export AIRFLOW_HOME=/path/to/airflow/home
$ airflow standalone
```

ref: https://github.com/apache/airflow/discussions/35903