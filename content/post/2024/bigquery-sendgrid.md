---
title: "BigQuery+SendGridでマーケティングオートメーションを構築する"
date: 2024-02-01
categories:
- データ基盤
---

BigQueryとSendGridを組み合わせて、顧客データを基にパーソナライズしたメールを送信する方法を解説します。  
送信したメールの効果測定・分析方法についても少し触れます。


## 1. SendGridテンプレートの作成

[Sendgrid公式ドキュメント](https://sendgrid.kke.co.jp/docs/Tutorials/A_Transaction_Mail/using_dynamic_templates.html#-Edit)を参考に、Dyamic templateを作成します。

今回は以下のようなメールを送りたいとします。

---
A様への今週のおすすめ商品です！

1. 商品B (1000円)
2. 商品C (1500円)

購入はこちらから！
https://example.com

---

Dynamic templateは以下のように記述します。

```text
{{customer_name}}様への今週のおすすめ商品です！

{{#each products}}
{{this.number}}. {{this.product_name}} ({{this.price}}円)
{{/each}}

購入はこちらから！
https://example.com
```

このテンプレートに対して、送信対象ごとに以下のようなデータを埋め込む事が目標になります。

```json
{
    "customer_name": "A",
    "products": [
        {
            "number": 1,
            "product_name": "商品B",
            "price": 1000
        },
        {
            "number": 1,
            "product_name": "商品C",
            "price": 1500
        },
    ]
}
```

## 2. 実装


BigQueryテーブル `mydataset.mytable` に顧客へのリコメンドデータが入っているとします。  
本稿では割愛しますが、BigQuery MLを用いれば簡単にリコメンドデータを生成することができます。  

---
|email|customer_name|number|product_name|price|
|-|-|-|-|-|
|a@example.com|A|1|商品B|1000|
|a@example.com|A|2|商品C|1500|
---

このリコメンド情報をDynamic templateに埋め込んでメールを送信するサンプルコードです。  
実際に運用する場合は例外処理を追加し、ワークフローエンジンで実行することになります。

`main.py`
```python
from google.cloud import bigquery
import sendgrid
from sendgrid.helpers.mail import Mail, To
import os
import json

SENDGRID_API_KEY = os.getenv('SENDGRID_API_KEY')
SENDGRID_TEMPLATE_ID = os.getenv('SENDGRID_TEMPLATE_ID')
sendgrid_client = sendgrid.SendGridAPIClient(SENDGRID_API_KEY)
bigquery_client = bigquery.Client()

query = """
SELECT
  email,
  ANY_VALUE(customer_name) customer_name,
  ARRAY_AGG(
    STRUCT(
      number,
      product_name,
      price
    )
    ORDER BY number
  ) AS products
FROM
  mydataset.mytable
GROUP BY email
"""

query_job = bigquery_client.query(query)
emails = [
    To(
        email=row["email"],
        dynamic_template_data=dict(row.items()) # (*1)クエリ結果を辞書型にしてDynamic templateに渡す
    ) for row in query_job
]

message = Mail(
    from_email=("sender-address@example.com", "Sender name"),
    to_emails=emails,
    is_multiple=True)
message.template_id = SENDGRID_TEMPLATE_ID

response = sendgrid_client.send(message)
x_message_id = response.headers["X-Message-Id"]
print(x_message_id) # (*2)このIDが分析に必須となるため、必要に応じてBigQueryにINSERTする等の実装を追加する
```

## 3. イベント収集

SendGridはメールが開封されたり、文中の外部リンクがクリックされたりした際にWebhookでイベントを送って来る事が可能です。  
https://sendgrid.kke.co.jp/docs/API_Reference/Webhooks/event.html

このイベントをPub/Sub等で受け取り、BigQueryに入れる仕組みを用意しておきます。

> POSTされるJSONデータに[smtp-id](https://sendgrid.kke.co.jp/docs/API_Reference/Webhooks/event.html#-Event-POST-Example)というプロパティがありますが、BigQueryではカラム名に `-` が使えません。  
場合によってはCloudFunctions等でアンダースコアに置換するなどの処理が必要になります。

## 4. 分析

[Pythonコード](#2-実装)の最後に登場した `X-Message-Id`ですが、 [イベント収集](#3-イベント収集)のデータに含まれる `sg_message_id` と以下のような関係があります。

<img src="https://support.sendgrid.kke.co.jp/hc/article_attachments/4404346672921/MessageID.png" width="640"/>

従って、とあるバッチで送信されたメールに関するイベントは以下のようなクエリで抽出することが可能です。

```sql
SELECT
  *
FROM
  mydataset.sendgrid_webhook_event
WHERE
  SPLIT(sg_message_id,".")[SAFE_OFFSET(0)] = 'X-Message-Id value'
```

この仕組みを利用して開封率、クリック率、Unsubscribe数などをモニタリングしつつ改善プロセスを回していきます。
