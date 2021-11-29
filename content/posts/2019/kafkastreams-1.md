---
title: "Kafka Streams DSLを一通り体験する(1. 準備編)"
date: 2019-06-04
draft: false
categories:
- java
- kafka
---
 
Kafka Streamsを使ってステートフルなストリーム処理を実装したいと思い立ったものの、[Kafka Streams Developer guide](https://kafka.apache.org/documentation/streams/developer-guide/)を読んでもいまいちよくわからなかったため、自分で一通り試してみました。

この記事では`Aggregate` `Reduce` `Join` `Windowing`など、Kafka Streams DSLでできる事を順番にテストし、挙動を確認していきます。また、`kafka-streams-test-utils`を用いたJUnitの実装についても解説します。

## 開発環境
- OpenJDK 11
- Maven 3.6
- Kafka 2.1.1

## 下準備

### プロジェクトの作成
以下の依存関係を追加します
- kafka-streams
- kafka-streams-test-utils
- junit-jupiter-api
- junit-jupiter-engine
- maven-surefire-plugin

```xml:pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>aaaanwz</groupId>
	<artifactId>kstream-dsl-test</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<properties>
		<maven.compiler.source>11</maven.compiler.source>
		<maven.compiler.target>11</maven.compiler.target>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka-streams</artifactId>
			<version>2.2.1</version>
		</dependency>
		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka-streams-test-utils</artifactId>
			<version>2.2.1</version>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.junit.jupiter</groupId>
			<artifactId>junit-jupiter-api</artifactId>
			<version>5.4.0</version>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.junit.jupiter</groupId>
			<artifactId>junit-jupiter-engine</artifactId>
			<version>5.4.0</version>
			<scope>test</scope>
		</dependency>
	</dependencies>
	<build>
		<pluginManagement>
			<plugins>
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-surefire-plugin</artifactId>
					<version>3.0.0-M3</version>
				</plugin>
			</plugins>
		</pluginManagement>
	</build>
</project>
```

##  Mainクラスの作成

まずはじめに、`input-topic`に入ってきたレコードをそのまま`output-topic`に送るストリーム処理(Topology)を実装します。

```java:src/main/java/core/Main.java
package core;

import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.Topology;

public class Main {

  public static void main(String[] args) {
    
  }

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();
    
    builder
    .stream("input-topic") //'input-topic'からレコードを取得します
    .to("output-topic"); //'output-topic'にレコードを書き込みます

    return builder.build();
  }

}
```

## JUnitテストの作成

Main#getTopology()の挙動を確認するJUnitを実装します。
`kafka-streams-test-utils`を用いると、実際のKafka環境がなくてもテストが可能です。

```java:src/main/test/java/core/MainTest.java
package core;

import java.util.Properties;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.TopologyTestDriver;
import org.apache.kafka.streams.test.ConsumerRecordFactory;
import org.junit.jupiter.api.Test;

class MainTest {
  /**
   * Kafkaへの接続情報。
   */
  private static final Properties properties;
  static {
    properties = new Properties();
    properties.put(StreamsConfig.APPLICATION_ID_CONFIG, "kafka-dsl-test");
    properties.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:1234");//実際のKafkaには接続しないため、適当な値でOK
    properties.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    properties.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
  }

  /**
   * TestDriverの準備
   */
  private static TopologyTestDriver testDriver;

  @BeforeEach
  void before() {
    testDriver = new TopologyTestDriver(Main.getTopology(), properties);
  }

  @AfterEach
  void after() {
    testDriver.close();
  }

  /**
   * 入力レコードをシリアライズするFactoryを使用し、TopologyTestDriverのinput-topicに入力します。
   * ConsumerRecordFactoryインスタンスは都度生成しないと内部のタイムスタンプが更新されず、後々のステートフルな処理がうまく動作しないため注意が必要です。
   */
  private void inputRecord(String topicName, String key,String value) {
    testDriver.pipeInput(
    new ConsumerRecordFactory<String,String>(Serdes.String().serializer(),Serdes.String().serializer()).create(topicName,key,value)
    );
  }
  
  /*
   * TopologyTestDriverのoutput-topicのレコードを取得し、Stringにデシリアライズします
   */
  private ProducerRecord<String,String> getOutputRecord(String topicName){
    return testDriver.readOutput(topicName,Serdes.String().deserializer(),Serdes.String().deserializer());
  }

  @Test
  void test() {
    inputRecord("input-topic", "key1", "ほげほげ");
    ProducerRecord<String, String> output = getOutputRecord("output-topic");
    System.out.println(output);//'output-topic'の内容をコンソールに表示します。
  }

}

```

`mvn test`で実行します。以下がコンソールに出力されましたでしょうか？

`ProducerRecord(topic=output-topic, partition=null, headers=RecordHeaders(headers = [], isReadOnly = false), key=key1, value=ほげほげ, timestamp=1559637393116)`
レコードのtopic, key, value, timestampがそれぞれ確認できますね。

問題なさそうであれば、本物のKafkaで動作させてみましょう。 `Main#main()`に追記します。
`BOOTSTRAP_SERVER_CONFIG`の値`localhost:9092`は各自の環境に合わせて置き換えてください。  

```java:src/main/java/core/Main.java
package core;

import java.util.Properties;
import java.util.concurrent.CountDownLatch;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.Topology;

public class Main {
  /**
   * Kafkaへの接続情報。
   */
  private static final Properties properties;
  static {
    properties = new Properties();
    properties.put(StreamsConfig.APPLICATION_ID_CONFIG, "kafka-dsl");
    properties.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    properties.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    properties.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,
        Serdes.String().getClass().getName());
  }

  public static void main(String[] args) throws InterruptedException {
    KafkaStreams streams =
        new KafkaStreams(getTopology(),properties);
    
    streams.setUncaughtExceptionHandler((Thread t, Throwable e) -> {
      e.printStackTrace();
      System.exit(1);
    });

    final CountDownLatch latch = new CountDownLatch(1);
    Runtime.getRuntime().addShutdownHook(new Thread("streams-shutdown-hook") {
      @Override
      public void run() {
        streams.close();
        latch.countDown();
      }
    });
    streams.start();
    latch.await();
  }

  public static Topology getTopology() {
    StreamsBuilder builder = new StreamsBuilder();
    
    builder
    .stream("input-topic") //topic名'input-topic'からレコードを取得します
    .to("output-topic"); //topic名 'output-topic'にレコードを書き込みます

    return builder.build();
  }

}
```

実行できましたら、`kafka-console-producer` と`kafka-console-consumer`で動作を確認してみてください。([参考:Kafka Quickstart](https://kafka.apache.org/quickstart))
`input-topic`に送ったレコードと同じ内容を`output-topic`から受け取れるかと思います。

[次回](https://qiita.com/aaaanwz/items/4695bb9439a2b0b9b898)は`main#getTopology()`に様々な関数を追加し、挙動をチェックしていきます。
