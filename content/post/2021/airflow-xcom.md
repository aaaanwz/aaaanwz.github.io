---
title: "Airflowで後続のOperatorに配列を渡す"
date: 2021-05-20
draft: false
categories:
- terraform
---

Apache AirflowにおいてOperator間で値を渡すにはXCOMを使用しますが、

- Airflow macroで文字列として取得する
- PythonOperatorでtask_instanceから取得する

の2通りの方法があります。

しかし、例えば

`GoogleCloudStorageListOperator`でファイルのリストを取得 >> `GoogleCloudStorageToBigQueryOperator` でリストされたファイルをBigQueryにロードする

といったことをやりたい場合、XCOMからファイルのリストを配列として取得しコンストラクタに渡さなければならないためすこし工夫が必要になります。
本稿ではその実装について記載します。

# NG

```python
...

list_files = GoogleCloudStorageListOperator(
    task_id='list_files',
    bucket='my_bucket',
    prefix='path/to/file/', 
    xcom_push=True,
    dag=dag
)

gcs_to_bigquery = GoogleCloudStorageToBigQueryOperator(
    task_id='gcs_to_bigquery',
    bucket='my_bucket',
    source_objects="{{ ti.xcom_pull(task_ids='list_files') }}",
    destination_project_dataset_table='project:dataset.table',
    autodetect=True,
    dag=dag
)

list_files >> gcs_to_bigquery

...
```

ファイル名の配列がデシリアライズされた状態で `source_objects` に渡されてしまうため動作しません。

# OK

```python
...

class CustomGcsToBigQueryOperator(GoogleCloudStorageToBigQueryOperator):
    def __init__(self, *args, **kwargs):
        super().__init__(
            source_objects=[],
            *args,
            **kwargs
            )
        self.source_objects_task_id = kwargs['source_objects_task_id']

    def execute(self, context):
        self.source_objects=context['ti'].xcom_pull(task_ids=self.source_objects_task_id)
        super().execute(context)

list_files = GoogleCloudStorageListOperator(
    task_id='list_files',
    bucket='my_bucket',
    prefix='path/to/file/', 
    xcom_push=True,
    dag=dag
)

gcs_to_bigquery = CustomGcsToBigQueryOperator(
    task_id='gcs_to_bigquery',
    bucket='my_bucket',
    source_objects_task_id='list_files',
    destination_project_dataset_table='project:dataset.table',
    autodetect=True,
    dag=dag
)

list_files >> gcs_to_bigquery

...
```

`GoogleCloudStorageToBigQueryOperator` を継承したカスタムオペレーターを実装し、`execute` で `xcom_pull` すればOKです。
