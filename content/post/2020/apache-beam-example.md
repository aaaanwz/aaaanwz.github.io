---
title: "Apache beam サンプルコード"
date: 2020-02-01
draft: false
categories:
- Data-pipeline
- Java
---

Apache beamの[Java quickstart](https://beam.apache.org/get-started/quickstart-java/)がいまいち分かりづらかったため、最小コードとデプロイ手順(Google Cloud Dataflow, AWS EMR)を備忘録としてまとめる

# WordCountサンプル
https://github.com/aaaanwz/beam-wordcount-sample

```
.
├── pom.xml
└── src
    └── main
        └── java
            ├── core
            │   └── WordCount.java
            └── dafn
                ├── ExtractWordsFn.java
                └── FormatAsTextFn.java
```

```xml:pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>aaaanwz</groupId>
  <artifactId>beam-wordcount-sample</artifactId>
  <version>0.1</version>
  <packaging>jar</packaging>
  <properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <beam.version>2.16.0</beam.version>
  </properties>
    <profiles>
      <profile>
        <id>direct-runner</id>
        <activation>
          <activeByDefault>true</activeByDefault>
        </activation>
        <dependencies>
          <dependency>
            <groupId>org.apache.beam</groupId>
            <artifactId>beam-runners-direct-java</artifactId>
            <version>${beam.version}</version>
            <scope>runtime</scope>
          </dependency>
        </dependencies>
      </profile>
      <profile>
      <id>dataflow-runner</id>
      <dependencies>
        <dependency>
          <groupId>org.apache.beam</groupId>
          <artifactId>beam-runners-google-cloud-dataflow-java</artifactId>
          <version>${beam.version}</version>
          <scope>runtime</scope>
        </dependency>
      </dependencies>
    </profile>
    <profile>
      <id>flink-runner</id>
      <dependencies>
        <dependency>
          <groupId>org.apache.beam</groupId>
          <artifactId>beam-runners-flink-1.9</artifactId>
          <version>${beam.version}</version>
          <scope>runtime</scope>
        </dependency>
      </dependencies>
    </profile>
  </profiles>
  <dependencies>
    <dependency>
      <groupId>org.apache.beam</groupId>
      <artifactId>beam-sdks-java-core</artifactId>
      <version>${beam.version}</version>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-simple</artifactId>
      <version>1.7.30</version>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <configuration>
          <createDependencyReducedPom>false</createDependencyReducedPom>
          <filters>
            <filter>
              <artifact>*:*</artifact>
              <excludes>
                <exclude>META-INF/*.SF</exclude>
                <exclude>META-INF/*.DSA</exclude>
                <exclude>META-INF/*.RSA</exclude>
              </excludes>
            </filter>
          </filters>
        </configuration>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <shadedArtifactAttached>true</shadedArtifactAttached>
              <shadedClassifierName>shaded</shadedClassifierName>
              <transformers>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                    <mainClass>core.WordCount</mainClass>
                </transformer>
              </transformers>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>

```

```java:WordCount.java
package core;

import org.apache.beam.sdk.Pipeline;
import org.apache.beam.sdk.coders.StringUtf8Coder;
import org.apache.beam.sdk.io.TextIO;
import org.apache.beam.sdk.options.PipelineOptions;
import org.apache.beam.sdk.options.PipelineOptionsFactory;
import org.apache.beam.sdk.transforms.Count;
import org.apache.beam.sdk.transforms.Create;
import org.apache.beam.sdk.transforms.MapElements;
import org.apache.beam.sdk.transforms.ParDo;
import dafn.ExtractWordsFn;
import dafn.FormatAsTextFn;

public class WordCount {
    public static void main(String[] args) {
        PipelineOptions options = PipelineOptionsFactory.fromArgs(args).withValidation().create();
        Pipeline p = Pipeline.create(options);
        p.apply("ReadLines",
                Create.of("a a a a a b b b b b c c c ").withCoder(StringUtf8Coder.of()))
                .apply(ParDo.of(new ExtractWordsFn())).apply(Count.perElement())
                .apply(MapElements.via(new FormatAsTextFn()))
                .apply("WriteCounts", TextIO.write().to("counts"));

        p.run().waitUntilFinish();
    }
}

```

```java:ExtractWordsFn.java
package dafn;

import org.apache.beam.sdk.metrics.Counter;
import org.apache.beam.sdk.metrics.Distribution;
import org.apache.beam.sdk.metrics.Metrics;
import org.apache.beam.sdk.transforms.DoFn;

public class ExtractWordsFn extends DoFn<String, String> {

  private static final long serialVersionUID = 1L;
  private final Counter emptyLines = Metrics.counter(ExtractWordsFn.class, "emptyLines");
  private final Distribution lineLenDist =
      Metrics.distribution(ExtractWordsFn.class, "lineLenDistro");

  @ProcessElement
  public void processElement(@Element String element, OutputReceiver<String> receiver) {
    lineLenDist.update(element.length());
    if (element.trim().isEmpty()) {
      emptyLines.inc();
    }

    String[] words = element.split("[^\\p{L}]+", -1);

    for (String word : words) {
      if (!word.isEmpty()) {
        receiver.output(word);
      }
    }
  }
}
```

```java:FormatAsTextFn.java
package dafn;

import org.apache.beam.sdk.transforms.SimpleFunction;
import org.apache.beam.sdk.values.KV;

public class FormatAsTextFn extends SimpleFunction<KV<String, Long>, String> {

    private static final long serialVersionUID = 1L;

    @Override
    public String apply(KV<String, Long> input) {
        return input.getKey() + ": " + input.getValue();
    }
}
```

# デプロイ方法

## ローカル実行
```shell
mvn package
java -jar ./target/beam-wordcount-sample-0.1-shaded.jar
```

## Google Cloud Dataflow
- gcloud CLIのインストール・設定済みであることが前提

```shell
mvn package -Pdataflow-runner
java -jar ./target/beam-wordcount-sample-0.1-shaded.jar --runner=DataflowRunner --project=xxxx --tempLocation=gs://<YOUR_GCS_BUCKET>/temp/
```

## AWS EMR
- aws cliのインストール・設定済みであることが前提
- [Flink を使用してクラスターを作成する](https://docs.aws.amazon.com/ja_jp/emr/latest/ReleaseGuide/flink-create-cluster.html)

```shell
mvn package -Pflink-runner
scp -i ~/.ssh/keypair.pem ./target/beam-wordcount-sample-0.1-shaded.jar ec2-user@ec2-xxx-xxx-xxx:/home/hadoop
```

```:Webコンソール
JAR の場所:command-runner.jar
メインクラス:なし
引数:flink run -m yarn-cluster -yn 2 /home/hadoop/beam-wordcount-sample-0.1-shaded.jar --runner=FlinkRunner
失敗時の操作:次へ
```
([参考](https://docs.aws.amazon.com/ja_jp/emr/latest/ReleaseGuide/flink-jobs.html))
