---
title: "Kafka Streams DSLを一通り体験する (2. ステートレス処理実践編)"
date: 2019-06-05
draft: false
categories:
- java
- kafka
---

Kafka Streams DSLのうち、ステートレスな操作(`branch`,`map`,`merge`など)を実際に触り、動作を確認します。
また最後に、本稿で登場する関数を使用してストリーム処理のFizzBuzzを実装してみます。

[前回の記事(準備編)](https://qiita.com/aaaanwz/items/d17ae435f6cff14d243c)のプロジェクトが作成済みである事を前提とします。


## 実際にやってみる

### Filter
Java Streamの`ByPredicate`と同じと思って差し支えありません。Java StreamのPredicateと紛らわしいのでimport対象に注意しましょう。

key, valueを引数にbooleanを返し、falseの場合はレコードが除外されます。

```java:Main.class
import org.apache.kafka.streams.kstream.Predicate;
...

  private static Predicate<String, String> predicate = (key, value) -> value.startsWith("あ");

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();

    builder
        .stream("input-topic", Consumed.with(Serdes.String(), Serdes.String())) //使用するデシリアライザにStringを明示指定します
        .filter(predicate)
        .to("output-topic");

    return builder.build();
  }
  ```

ここで`Consumed.with(...)`が新たに登場しました。Predicateの引数がString型なので、デシリアライザも明示指定する必要があるためです。

また`.filter((key, value)-> value.startsWith("あ"))`のように直接ラムダ式を記述することももちろん可能です。

さて、テストを実行してみましょう。

```java:MainTest.class
  @Test
  void test() {
    inputRecord("input-topic", "key1", "あけまして");
    System.out.println(getOutputRecord("output-topic"));
    inputRecord("input-topic", "key1", "おめでとう");
    System.out.println(getOutputRecord("output-topic"));
  }
```
  

```text:実行結果(可読性のため編集しています)
topic=output-topic, key=key1, value=あけまして
null
```
"あ"から始まるvalueを持つレコードだけが`filter`を通過することが確認できました。

### Branch
Java関数型には今の所存在しない機能です。
複数の`kstream.Predicate`を引数に持ち、ストリームを分岐させることができます。  


```java:Main.class
  private static Predicate<String, String> predicateA = (key, value) -> value.startsWith("あ");
  private static Predicate<String, String> predicateB = (key, value) -> value.startsWith("い");

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();

    @SuppressWarnings("unchecked")
    KStream<String, String>[] branchedStream = builder
        .stream("input-topic", Consumed.with(Serdes.String(), Serdes.String()))
        .branch(predicateA, predicateB);

    branchedStream[0].to("output-topicA");
    branchedStream[1].to("output-topicB");

    return builder.build();
  }
```
branchを用いるとKStream型の配列を得ることができます。一番目のPredicate(predicateA)を通過したレコードは[0]、一番目は通過せず二番目(predicateB)を通過したPredicateが[1]に流れていきます。

```java:MainTest.class
  @Test
  void test() {
    inputRecord("input-topic", "key1", "あひる");
    inputRecord("input-topic", "key1", "いんこ");
    inputRecord("input-topic", "key1", "からす");
    System.out.println(getOutputRecord("output-topicA"));
    System.out.println(getOutputRecord("output-topicA"));
    System.out.println(getOutputRecord("output-topicB"));
    System.out.println(getOutputRecord("output-topicB"));
  }
```

```text:実行結果
topic=output-topicA, key=key1, value=あひる
null
topic=output-topicB, key=key1, value=いんこ
null
```

"あひる"はoutput-topicA、"いんこ"はoutput-topicB、"からす"は除外される事が確認できました。
この記事の後半に登場する`merge`と供に、非常に便利な機能です。

### Map
レコードの内容を変換する関数です。
key,valueを引数にとる`KeyValueMapper`、valueのみを引数にとる`ValueMapper`が用意されており、それぞれJava Streamの`BiFunction`、`Function`とほぼ同じです。

```java:Main.class
  private static KeyValueMapper<String, String, KeyValue<String, String>> keyValueMapper =
      (key, value) -> KeyValue.pair(key + "_hoge", value + "_fuga");

  private static ValueMapper<String, String> valueMapper = (value) -> value + "_foo";

  private static KeyValueMapper<String, String, String> keyMapper =
      (key, value) -> key + "_" + value + "_bar";

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();
    KStream<String, String> stream = builder
        .stream("input-topic", Consumed.with(Serdes.String(), Serdes.String()));

    stream
        .map(keyValueMapper)
        .to("output-topicA");

    stream
        .mapValues(valueMapper)
        .to("output-topicB");

    stream
        .selectKey(keyMapper)
        .to("output-topicC");

    return builder.build();
  }
```

ここでは、keyとvalueを両方変換(output-topicA)、valueのみ変換(output-topicB)、keyのみ変換(output-topicC)の3パターンを試してみます。
それぞれ`.map`,`.mapValues`,`.selectKey`で呼び出しますが、`.selectKey`の引数は`KeyValueMapper`なのが少し紛らわしいところです。

keyとvalueを両方変換する場合、KeyValueMapperの帰り値はKeyValue型を使用します。入力と異なる型を返す場合は終端処理`.to`でシリアライザを指定する必要があります。  
(例:`.to("output-topic",Produced.with(Serdes.Integer(),Serdes.Integer())`)

これまで`key`に関して何も触れて来ませんでしたが、ステートフルな処理で多用するためその折に解説します。またKafkaクラスタを組んだ際、レコードがどのノードに送られるかを決定する値でもあります。

```java:MainTest.class
  @Test
  void test() {
    inputRecord("input-topic", "key1", "value1");
    System.out.println(getOutputRecord("output-topicA"));
    System.out.println(getOutputRecord("output-topicB"));
    System.out.println(getOutputRecord("output-topicC"));
  }
```

```text:実行結果
topic=output-topicA, key=key1_hoge, value=value1_fuga
topic=output-topicB, key=key1, value=value1_foo
topic=output-topicC, key=key1_value1_bar, value=value1
```
key&value、valueのみ、keyのみの変換がそれぞれ行われている事が確認できました。

### FlatMap
Java Streamと同様にflatMapも用意されています。
1つのレコードをIterableな値に変換する事で、レコードごと複数に分割する機能です。

```java:Main.java
  private static ValueMapper<String, List<String>> valueMapper =
      (value) -> Arrays.asList(value.split(":"));

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();
    KStream<String, String> stream = builder
        .stream("input-topic", Consumed.with(Serdes.String(), Serdes.String()));

    stream
        .flatMapValues(valueMapper)
        .to("output-topic");

    return builder.build();
  }
```

```java:MainTest.java
  @Test
  void test() {
    inputRecord("input-topic", "key1", "あ:い:う");
    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
  }
```

```java:実行結果
topic=output-topic, key=key1, value=あ
topic=output-topic, key=key1, value=い
topic=output-topic, key=key1, value=う
```

### ForeachAction
Java Streamの`BiConsumer`と同等で、戻り値のない処理を行います。
`.peek`では、レコードをread onlyとして何らかの処理を行います。（主にロギングに使用されるかと思います)
`.forEach`は`.to`と同様の終端処理として機能します。
終端処理が実施されるとKafkaにおいてそのレコードのConsumeが完了したと見なされます。

```java:Main.class
  private static ForeachAction<String, String> foreachAction =
      (key, value) -> System.out.println("print (" + key + " : " + value + ")");

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();
    KStream<String, String> stream = builder
        .stream("input-topic", Consumed.with(Serdes.String(), Serdes.String()));

    stream
        .peek(foreachAction)
        .map((key, value) -> KeyValue.pair(key + "_changed", value + "_changed"))
        .foreach(foreachAction);

    return builder.build();
  }
```

```java:MainTest.class
  @Test
  void test() {
    inputRecord("input-topic", "key1", "ほげほげ");
  }
```

```text:実行結果
print (key1 : ほげほげ)
print (key1_changed : ほげほげ_changed)
```

レコードの内容がコンソールに出力されました。またforeachによってレコードのConsumeは完了し、Kafkaのオフセットが進行します。

### through
レコードの内容をトピックに書き込み、さらに後続処理に渡します。
処理途中のデータから別のストリームを開始したい場合に使用します。

```java:Main.class
  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();
    KStream<String, String> stream = builder
        .stream("input-topic", Consumed.with(Serdes.String(), Serdes.String()));

    stream
        .through("output-topicA")
        .map((key, value) -> KeyValue.pair(key + "_changed", value + "_changed"))
        .to("output-topicB");

    return builder.build();
  }
```

```java:MainTest.class
  @Test
  void test() {
    inputRecord("input-topic", "key1", "ほげほげ");
    System.out.println(getOutputRecord("output-topicA"));
    System.out.println(getOutputRecord("output-topicB"));
  }
```

```text:結果
topic=output-topicA,  key=key1, value=ほげほげ
topic=output-topicB,  key=key1_changed, value=ほげほげ_changed
```
1つのレコードが形を変え2つのトピックに送られました。

### merge
2つのストリームを1つに結合します。  
とりあえず実行してみましょう。

```java:Main.class
  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();
    KStream<String, String> streamA = builder
        .stream("input-topicA", Consumed.with(Serdes.String(), Serdes.String()));

    KStream<String, String> streamB = builder
        .stream("input-topicB", Consumed.with(Serdes.String(), Serdes.String()));

    streamA
        .merge(streamB)
        .to("output-topic");

    return builder.build();
  }
```

```java:MainTest.class
  @Test
  void test() {
    inputRecord("input-topicA", "key1", "ほげほげ");
    inputRecord("input-topicB", "key1", "ふがふが");
    System.out.println(getOutputRecord("output-topic"));
    System.out.println(getOutputRecord("output-topic"));
  }
```

```text:実行結果
topic=output-topic, key=key1, value=ほげほげ
topic=output-topic, key=key1, value=ふがふが
```

異なるトピックに入れたレコードが一つのストリームになりました。

## Kafka Streams FizzBuzzを実装する
最後に、これまでの知識を無駄に使って冗長なFizzBuzzを実装してみましょう。

```java:Main.class
  @SuppressWarnings("unchecked")
  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();

    KStream<Integer, String> inputStream = builder
        .stream("input-topic", Consumed.with(Serdes.String(), Serdes.Integer()))
        .map((key, value) -> KeyValue.pair(value, ""));
    
    KStream<Integer, String> branchFizz[] = inputStream
        .branch((key, value) -> key % 3 == 0,
            (key, value) -> true);

    KStream<Integer, String> branchBuzz[] = branchFizz[0]
        .mapValues(value->"Fizz")
        .merge(branchFizz[1])
        .branch((key, value) -> key % 5 == 0,
            (key, value) -> true);
    
    KStream<Integer, String> branchFizzBuzz[] = branchBuzz[0]
        .mapValues(value -> value + "Buzz")
        .merge(branchBuzz[1])
        .branch((key,value)->value.equals(""),
            (key,value)->true);
    
    branchFizzBuzz[0]
        .mapValues((key, value) -> String.valueOf(key))
        .merge(branchFizzBuzz[1])
        .to("output-topic", Produced.with(Serdes.Integer(), Serdes.String()));

    return builder.build();
  }
```

```java:MainTest.class
  private void inputRecord(String topicName, String key, Integer value) {
    testDriver.pipeInput(
        new ConsumerRecordFactory<String, Integer>(Serdes.String().serializer(),
            Serdes.Integer().serializer()).create(topicName, key, value));
  }

  private ProducerRecord<Integer, String> getOutputRecord(String topicName) {
    return testDriver.readOutput(topicName, Serdes.Integer().deserializer(),
        Serdes.String().deserializer());
  }

  @Test
  void test() {
    for (int i = 1; i <= 30; i++) {
      inputRecord("input-topic", "key1", i);
      System.out.println(getOutputRecord("output-topic").value());
    }
  }
```

いかがでしたでしょうか。  
[次回](https://qiita.com/aaaanwz/items/ae483ace5d7de17c4c11)はいよいよステートフルな処理を試してみようと思います。
