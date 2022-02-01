---
title: "Kinesis Data FirehoseからMongoDBにデータを流してみる"
date: 2022-01-31
draft: true
categories:
- その他
---

[Amazon Kinesis Data Firehose が MongoDB Cloud へのデータ配信のサポートを開始](https://aws.amazon.com/jp/about-aws/whats-new/2020/07/amazon-kinesis-data-firehose-supports-data-delivery-mongodb-cloud/) したそうなので試してみましたが、色々と「ん？」と思う点があったので書き残します。

詳細は [Using MongoDB Realm WebHooks with Amazon Kinesis Data Firehose](https://www.mongodb.com/developer/how-to/Realm-AWS-Kinesis-Firehose-Destination/) を見るようにとあります。

## MongoDB Atlas でClusterを作成

MongoDB Atlas、MongoDB Realmという単語が登場しますが、AtlasはDBaaSとしてのMongoDBそのもの、RealmはAtlasを操作するためのインターフェースとなるサービスのようです。
まずはAtlasでCluster, Database, Collectionを作成します。

- [Get Started with Atlas](https://docs.atlas.mongodb.com/getting-started/)

## APIキーを作成

- [Create a Server API Key](https://docs.mongodb.com/realm/authentication/api-key/#create-a-server-api-key)

## MongoDB Realm Functionsを実装

てっきりGUIポチポチで連携完了するものかとおもってましたが、FirehoseからのWebhookエンドポイントとなるサーバーレス関数を自前で実装する必要があります。  
これ、Kinesis Data Streams + Lambdaでよくね...？

- [MongoDB Realm Functions](https://docs.mongodb.com/realm/functions/#functions)

AWSのリリースノートからリンクされている[サンプルコード](https://www.mongodb.com/developer/how-to/Realm-AWS-Kinesis-Firehose-Destination/#the-realm-function)の冒頭はこんな感じ。

```js
exports = function(payload, response) {
    /* Using Buffer in Realm causes a severe performance hit
    this function is ~6 times faster
    */
    const decodeBase64 = (s) => {
        var e={},i,b=0,c,x,l=0,a,r='',w=String.fromCharCode,L=s.length
        var A="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
        for(i=0;i<64;i++){e[A.charAt(i)]=i}
        for(x=0;x<L;x++){
            c=e[s.charAt(x)];b=(b<<6)+c;l+=6
            while(l>=8){((a=(b>>>(l-=8))&0xff)||(x<(L-2)))&&(r+=w(a))}
        }
        return r
    }
...
```

自前でbase64デコードが実装されている事自体に驚きですが、マルチバイト文字に対応していないためこのままでは使えません。

幸いなことに [Import External Dependencies](https://docs.mongodb.com/realm/functions/import-external-dependencies/) で外部のnpmパッケージが使えます。
Function Editor画面で `Add Dependency`を選択し、`js-base64` の `3.7.2`を追加します。
importは非対応のようで、requireを使うことになります。

```js
const decoded = require("js-base64").decode(encoded)
```


最終的に以下のコードで関数を作成しました。
認証には先程作成したAPIキーを使いたいので、関数を作成する際に選択するAuthenticationは `System` にします。

```js
exports = (payload, response) => {
  const body = JSON.parse(payload.body.text())
  const requestId = body.requestId
  const timestamp = new Date(body.timestamp)
  const firehoseAccessKey = payload.headers["X-Amz-Firehose-Access-Key"]


  const bulkOp = context.services.get("mongodb-atlas").db("test-db").collection("test-collection").initializeOrderedBulkOp()
  body.records.forEach((record, index) => {
    data = JSON.parse(require("js-base64").decode(record.data))
    bulkOp.insert(data)
  })
  
  
  response.addHeader(
    "Content-Type",
    "application/json"
  )

  
  bulkOp.execute().then(() => {
    response.setStatusCode(200)
    response.setBody(JSON.stringify({
      requestId: requestId,
      timestamp: timestamp.getTime()
    }))
    return
  }).catch((error) => {
    response.setStatusCode(500)
    response.setBody(JSON.stringify({
      requestId: requestId,
      timestamp: timestamp.getTime(),
      errorMessage: error
    }))
    return
  })
}
```

## Realm HTTPS Endpointsを作成

リリースノートが書かれてからまだ半年ですが、[Creating our Realm WebHook](https://www.mongodb.com/developer/how-to/Realm-AWS-Kinesis-Firehose-Destination/#creating-our-realm-webhook) は既にDeprecatedになっています。

公式ドキュメントの[Configure HTTPS Endpoints](https://docs.mongodb.com/realm/endpoints/configure/) を参照し、`Add An Endpoint` を選択します。
先程作成したFunctionを選択し、HTTP Methodは `POST`、Request Validationは `No Additional Authorization` を選択します。

## Kinesis Data Firehoseを設定

[Choose MongoDB Cloud for Your Destination](https://docs.aws.amazon.com/firehose/latest/dev/create-destination.html#create-destination-mongodb) の通りにKinesis Data Firehoseを設定します。

## テスト

AWSマネジメントコンソールの先程作成したKinesis Delivery streamsから `Test with demo data`を選択し、テストデータを流してみます。
うまくいくとCollectionsにKinesisのテストデータが追加されています。
うまく行かない場合はMongoDB Cloud側の `LOGS`を確認します。
