---
title: "Kafka Streams DSLを一通り体験する (3. ステートフル処理実践編)"
date: 2019-06-07
draft: false
categories:
- java
- kafka
---

Kafka Streams DSLのうち、ステートフルな操作(`join`,`reduce`,`aggregate`,`windowing`など)を実際に触り、動作を確認します。

また最後に、本稿と[前回](https://qiita.com/aaaanwz/items/4695bb9439a2b0b9b898)で登場した関数を使用してステートフルなストリームFizzBuzzを実装してみます。  

## 実際にやってみる
[前々回の記事(準備編)](https://qiita.com/aaaanwz/items/d17ae435f6cff14d243c)のプロジェクトが作成済みである事を前提とします。


### KTable
まずはじめに、`KTable`,`KGroupedStream`について知っておく必要があります。
`KGroupedStream`はkeyの値毎にグループ化された`KStream`で、`KTable`はkeyとvalueの最新状態を保持するテーブルとして扱えるものです。  
KTableは`new StreamsBuilder().table("topic-name")...`のように直接トピックから生成したり、`KGroupedStream`を集約して生成したりと様々なルートで生成することができます。  
公式ドキュメントの以下の図が非常に分かりやすいです。

![](https://kafka.apache.org/20/images/streams-stateful_operations.png)
[画像リンク元ページ](https://kafka.apache.org/20/documentation/streams/developer-guide/dsl-api.html#id11)


### Aggregate
`KGroupedStream`をkeyごとに集約し、`KTable`に変換します。
コードと実行結果を見るのが一番早いと思います。

```java:Main.class
  private static Initializer<String> initializer = () -> "InitVal";

  private static Aggregator<String, String, String> aggregator =
      (key, val, agg) -> agg + " & " + val;

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();

    builder
        .stream("input-topic")
        .groupByKey()
        .aggregate(initializer, aggregator)
        .toStream() // KTableの更新履歴をストリームとして取り出す
        .to("output-topic");

    return builder.build();
  }
```
`Initializer`は初期値を返す関数で、Java streamで言うと`Supplier`です。  
`Aggregator`は`key`,`value`,`現在のステート`の3つを引数として受け取り、新たなステートを生成するための関数です。
`aggregate`の結果はKTableですが、更新結果を確認するために`toStream()`を使います。

```java:MainTest.class
  @Test
  void test() throws InterruptedException {
    inputRecord("input-topic", "key1", "hoge");
    inputRecord("input-topic", "key1", "fuga");
    inputRecord("input-topic", "key2", "foo");
    inputRecord("input-topic", "key2", "bar");

    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
  }
```

```text:実行結果(可読性のため編集しています)
topic=output-topic, key=key1, value=InitVal & hoge
topic=output-topic, key=key1, value=InitVal & hoge & fuga
topic=output-topic, key=key2, value=InitVal & foo
topic=output-topic, key=key2, value=InitVal & foo & bar
```

keyに対応するvalueが、aggregatorに記述した通りにどんどん連結されていっていますね。

簡易版のような機能として`reduce`と`count`も用意されています。
reduceはinitializerが無いaggregateのようなもので、特定のkeyに対して最初に来たvalueが初期値になります。

```java:Main.class
private static Reducer<String> reducer = (val, agg) -> agg + " & " + val;

...

  builder
    .stream("input-topic")
    .groupByKey()
    .reduce(reducer)
    .toStream()
    .to("output-topic");

...
```

```text:実行結果
key=key1, value=hoge
key=key1, value=hoge & fuga
```

### count
簡易版aggregateで、keyに対するレコード数をカウントするKTableを生成します。  
元のレコード内容は失われてしまうので、`through`などで別のストリームを生やして使うのが一般的かと思います。

```java:Main.class
  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();

    builder
        .stream("input-topic");
        .groupByKey()
        .count()
        .toStream()
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));

    return builder.build();
  }
```

```java:MainTest.class
  private ProducerRecord<String, Long> getOutputRecord(String topicName) {
    return testDriver.readOutput(topicName, Serdes.String().deserializer(),
        Serdes.Long().deserializer());
  }

  @Test
  void test() throws InterruptedException {
    inputRecord("input-topic", "key1", "hoge");
    inputRecord("input-topic", "key1", "fuga");
    inputRecord("input-topic", "key2", "foo");
    inputRecord("input-topic", "key2", "bar");

    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
  }
```

```text:実行結果
topic=output-topic, key=key1, value=1
topic=output-topic, key=key1, value=2
topic=output-topic, key=key2, value=1
topic=output-topic, key=key2, value=2
```

### Windowing
ストリームを時間枠で切り取ります。Tubling, Hopping, Sliding, Sessionの4種類がありここでは紹介しきれないため、[公式ドキュメント](https://docs.confluent.io/current/streams/developer-guide/dsl-api.html#windowing)の図を見ていただくのが最もわかりやすいかと思います。

今回は1000msのTumblingでレコードをcountしてみます。
`windowedBy`を行うと、`Windowed`オブジェクトがkeyとして使用されます。
試しに`.toString()`でkeyの内容も確認してみましょう

```java:Main.class
  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();

    builder
        .stream("input-topic")
        .groupByKey()
        .windowedBy(TimeWindows.of(Duration.ofMillis(1000)))
        .count()
        .toStream()
        .selectKey((key, value) -> key.toString())
        .to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));

    return builder.build();
  }
```

```java:MainTest.class
  @Test
  void test() throws InterruptedException {
    inputRecord("input-topic", "key1", "hoge");
    inputRecord("input-topic", "key1", "fuga");
    inputRecord("input-topic", "key1", "foo");
    Thread.sleep(1000);
    inputRecord("input-topic", "key1", "bar");

    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
  }
```

```text:実行結果
topic=output-topic, key=[key1@1559886985000/1559886986000], value=1
topic=output-topic, key=[key1@1559886985000/1559886986000], value=2
topic=output-topic, key=[key1@1559886985000/1559886986000], value=3
topic=output-topic, key=[key1@1559886986000/1559886987000], value=1
```

keyが`{元のkey値}@{window開始時刻}/{window終了時刻}`になっていますね。
そして`hoge``fuga``foo`は同一window内でカウントされ、時間が開いた`bar`は次のwindowに入っています。

### Join

同一のkeyを持つ2つのレコードを組み合わせ、新たなストリームを生成します。
KStream同士、KStreamとKTable、KTable同士それぞれで実行できます。
例としてKStream同士のjoinを行ってみます。

```java:Main.class
  private static ValueJoiner<String, String, String> joiner = (left, right) -> left + " & " + right;

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();

    KStream<String, String> streamA = builder.stream("input-topicA");
    KStream<String, String> streamB = builder.stream("input-topicB");    
        
    streamA
        .join(streamB, joiner, JoinWindows.of(Duration.ofMillis(100)))
        .to("output-topic");

    return builder.build();
  }
```

`ValueJoiner`は2つのvalueから新たなvalueを生成する関数です。

`KStream`同士をjoinする場合は、`JoinWindows`で2つのレコードが揃うまでの待ち受け時間を定義します。相手がKTableの場合はkeyに対する最後(最新)のvalueの値がjoinされるため、Windowの定義は不要です。

```java:MainTest.class
  @Test
  void test() throws InterruptedException {
    inputRecord("input-topicA", "key1", "hoge");
    inputRecord("input-topicB", "key1", "fuga");
    inputRecord("input-topicA", "key2", "foo");
    Thread.sleep(200);
    inputRecord("input-topicB", "key2", "bar");

    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
  }
```

```text:実行結果
topic=output-topic, key=key1, value=hoge & fuga
null
```
`key1`を持つ`hoge`と`fuga`が結合しています。また、`foo`と`bar`はJoinWindow内に収まっていないため、joinが実施されていません。

`join`の他に`leftJoin`と`outerJoin`が用意されています。`KSQL`同様、リレーショナルDBの感覚ですね。


## ステートフルFizzBuzzを実装する

さて、長くなりましたがKafka Streams DSLの大方を触ってみました。  
これまで見てきた物を使って、ステートフルなFizzBuzzを実装してみましょう。
以下の要件でやってみます。

- 自然数をconsumeする度に加算していき、総和が3の倍数ならFizz, 5の倍数ならBuzz, 15の倍数ならFizzBuzz、それ以外なら現在の総和をoutput-topicにproduceする。
- FizzBuzzが出力されるか、5秒以上入力レコードが来なければ総和を0に戻す

```java:Main.class
  private static Aggregator<String, Integer, String> aggregator = (k, v, a) -> {
    Integer sum;
    try {
      sum = Integer.valueOf(a);
    } catch (NumberFormatException e) {
      sum = 0;
    }
    sum += v;
    if (sum % 15 == 0) {
      return "FizzBuzz (" + sum + ")";
    } else if (sum % 3 == 0) {
      return "Fizz (" + sum + ")";
    } else if (sum % 5 == 0) {
      return "Buzz (" + sum + ")";
    }
    return sum.toString();
  };

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();

    builder
        .stream("input-topic", Consumed.with(Serdes.String(), Serdes.Integer()))
        .groupByKey(Grouped.with(Serdes.String(), Serdes.Integer()))
        .windowedBy(TimeWindows.of(Duration.ofMillis(5000)))
        .aggregate(() -> "0", aggregator)
        .toStream()
        .selectKey((k, v) -> k.toString())
        .to("output-topic", Produced.with(Serdes.String(), Serdes.String()));

    return builder.build();
  }
  ```

あまり美しくはないですが、最も単純に「前回の計算結果をそのままステートにする」実装にしてみました。
是非実際のKafka環境をセットアップして遊んでみてください。

Consume/Publishで実装するとちょっと面倒な要件も、非常にシンプルに記述できることが実感できたかと思います。
これで完結となります、ご覧いただきありがとうございました！
