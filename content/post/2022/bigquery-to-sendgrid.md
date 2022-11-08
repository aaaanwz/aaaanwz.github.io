---
title: "BigQueryのデータを元にパーソナライズされたメールを送信する"
date: 2022-11-08
categories:
- BigQuery
- BI・MA
---

BigQueryに格納されている行動ログを元にマーケティングオートメーションを構築、SendGridでメールを発射したいケースは多々あるかと思います。  
MAツールはかなり高価なものが多いため、シュッと自前構築するためのサンプルコードを掲載します。  
送信したメールの効果測定・分析方法についても少し触れます。


## 1. SendGridテンプレートの作成

[公式ドキュメント](https://sendgrid.kke.co.jp/docs/Tutorials/A_Transaction_Mail/using_dynamic_templates.html#-Edit)を参考に、Dyamic templateを作成します。

詳細は割愛しますが、今回は以下のようなメールを送りたいとします。

```
foobar様への今週のおすすめ商品です！

1. ほげほげ (1000円)
2. ふがふが (1500円)

購入はこちらから！
https://example.com
```

テンプレートはこんな感じで記述します。

```
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
    "customer_name": "foobar",
    "products": [
        {
            "number": 1,
            "product_name": "ほげほげ",
            "price": 1000
        },
        {
            "number": 1,
            "product_name": "ふがふが",
            "price": 1500
        },
    ]
}
```

## 2. 実装

環境変数でSQLを受け取り、クエリ結果をDynamic templateに埋め込んでメールを発射するコードです。
実際に運用する場合は例外処理諸々を追加してください。

`main.py`

```python
from google.cloud import bigquery
import sendgrid
from sendgrid.helpers.mail import Mail, To
import os
import json

SENDGRID_API_KEY = os.getenv('SENDGRID_API_KEY')
SENDGRID_TEMPLATE_ID = os.getenv('SENDGRID_TEMPLATE_ID')
QUERY = os.getenv('QUERY')

sendgrid_client = sendgrid.SendGridAPIClient(SENDGRID_API_KEY)
bigquery_client = bigquery.Client()

query_job = bigquery_client.query(QUERY)
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
print(x_message_id) # (*2)このIDが分析に必須となるため、必要に応じてBigQueryに貯める等の実装を追加する
```

BigQueryテーブル `mydataset.mytable` に顧客へのリコメンド情報が入っているとします。  
(実際こんな構造になっていることは無いと思いますが、説明の都合です)

|email|customer_name|number|product_name|price|
|-|-|-|-|-|
|foobar@example.com|foobar|1|ほげほげ|1000|
|foobar@example.com|foobar|2|ふがふが|1500|

環境変数 `QUERY` に渡すSQLは以下のような感じです。

```sql
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
```

上記コードをAirflowなどで運用していきます。

## 3. イベント収集

SendGridはメールが開封されたり、文中の外部リンクがクリックされたりした際にWebhookでイベントを送って来る事が可能です。
https://sendgrid.kke.co.jp/docs/API_Reference/Webhooks/event.html

このイベントを受け取り、BigQueryに入れる仕組みを用意しておきます。
こちらのコードは割愛しますが、CloudFunctionsや[fluentd http input plugin](https://docs.fluentd.org/input/http)などでPOSTされてきたデータをそのままBigQueryテーブルにSTREAMING INSERTされるようにします。

落とし穴としてキーに[smtp-id](https://sendgrid.kke.co.jp/docs/API_Reference/Webhooks/event.html#-Event-POST-Example)という値がありますがBigQueryではカラム名に `-` が使えないため、場合によってはTransformステップでアンダースコアに置換するなどの処理が必要になります。

## 4. 分析

[Pythonコード](#2-実装)の最後に登場した `X-Message-Id`ですが、 [イベント収集](#3-イベント収集)のデータに含まれる `sg_message_id` と以下のような関係があります。

![](https://support.sendgrid.kke.co.jp/hc/article_attachments/4404346672921/MessageID.png)
[公式ドキュメント](https://support.sendgrid.kke.co.jp/hc/ja/articles/360000222542-%E9%80%81%E4%BF%A1%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88%E3%81%AE%E5%86%85%E5%AE%B9%E3%81%A8%E3%82%A4%E3%83%99%E3%83%B3%E3%83%88%E3%82%92%E7%B4%90%E4%BB%98%E3%81%91%E3%82%8B%E6%96%B9%E6%B3%95%E3%82%92%E6%95%99%E3%81%88%E3%81%A6%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84)

従って、とあるバッチで送信されたメールに関するイベントは以下のようなクエリで抽出することが可能です。

```sql
SELECT
  *
FROM
  mydataset.sendgrid_webhook_event
WHERE
  SPLIT(sg_message_id,".")[SAFE_OFFSET(0)] = 'X-Message-Id value'
```

これを利用して開封率、クリック率、Unsubscribe数などをモニタリングしつつ改善プロセスを回していきます。

## まとめ

- BigQueryとSendGrid Dynamic Templateを組み合わせて、パーソナライズされたメールの自動送信を簡単に実現できる
  - BigQuery MLと組み合わせると機械学習の導入も一瞬
- Webhook経由でメールの開封、URLクリックなどのイベントデータが収集できる
- メール送信時のレスポンスヘッダに含まれる `X-Message-Id` はイベントデータの絞り込みを行う際に必要になる