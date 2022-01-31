---
title: "Kinesis Data FirehoseからMongoDBにINSERTする際の罠"
date: 2022-01-31
draft: true
categories:
- その他
---

[Amazon Kinesis Data Firehose が MongoDB Cloud へのデータ配信のサポートを開始](https://aws.amazon.com/jp/about-aws/whats-new/2020/07/amazon-kinesis-data-firehose-supports-data-delivery-mongodb-cloud/) したそうなので試してみましたが、色々と「ん？」と思う点があったので書き残します。

詳細は [Using MongoDB Realm WebHooks with Amazon Kinesis Data Firehose](https://www.mongodb.com/developer/how-to/Realm-AWS-Kinesis-Firehose-Destination/) を見るようにとあります。

大きな流れとしては、

1. MongoDB Atlas Database, Collectionを作成
2. MongoDB Realm Functionsを実装
3. Realm HTTPS Endpointsを作成
4. Kinesis Data Firehoseを設定

になります。

## 3rd Party ServicesがDeprecatedになっている

↑の記事が書かれてからまだ半年ですが、[Creating our Realm WebHook](https://www.mongodb.com/developer/how-to/Realm-AWS-Kinesis-Firehose-Destination/#creating-our-realm-webhook) はDeprecatedになっています。
公式ドキュメントの[Configure HTTPS Endpoints](https://docs.mongodb.com/realm/endpoints/configure/) を参照します。

## Realm Functionを自前で実装する必要がある

てっきりGUIポチポチで連携完了するものかとおもってましたが、FirehoseからのWebhookエンドポイントとなるサーバーレス関数を自前で実装する必要があります。  
これ、Kinesis Data Streams + Lambdaでよくね...？

しかもサンプルコードの冒頭はこんな感じ。

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

Base64デコードを自前で実装しているのも驚きですが、[A-Za-z1-9+/]以外の文字はガン無視とは。

幸いなことに [Import External Dependencies](https://docs.mongodb.com/realm/functions/import-external-dependencies/) で外部のnpmパッケージが使えます。
importは非対応のようで、requireを使うことになります。

```js
const decoded = require("js-base64").decode(encoded)
```

