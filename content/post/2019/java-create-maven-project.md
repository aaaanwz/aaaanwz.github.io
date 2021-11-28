---
title: "Javaプロジェクト作成からCIOpsまで"
date: 2019-07-24
draft: false
categories:
- java
- kubernetes
---

git branchに変更が加わった際、

- JUnit test (with MySQL)
- docker build
- Kubernetes環境にデプロイ (CIOps)

が行われるJavaプロジェクトを構築します。

> 本番運用ではArgoCDなどgitOps構築をお勧めします

## 登場するもの

### OSS
- Maven
- MySQL
- Docker

### サービス
- GitHub
- CircleCI
- AWS (ECR, EKS) ⇦ 微修正でその他マネージドk8sにも応用可能かと思います。

# サンプルプロジェクトの実装
最終的にディレクトリ構成はこんな感じになります。順を追って作っていきます。
[GitHub](https://github.com/aaaanwz/java-autodeploy-example)からcloneして頂いても結構です。

```
testproject/
    ├ src/
    │   ├ main/
    │   │   └ java/
    │   │        └testpackage/
    │   │            └Main.java
    │   └ test/
    │       └ java/
    │           └testpackage/  
    │              └MainTest.java   
    ├ .cifcleci/
    │   └ config.yml
    ├ schema.sql
    ├ pom.xml
    ├ deploy.yaml
    └ Dockerfile

```

## 1. スキーマ定義

テーブルスキーマを記述します。resourceフォルダに置いても良いのですが、今回はトップディレクトリに配置することにします。  
データベース名は指定しません。

```sql:schema.sql 
CREATE TABLE user (id int, name varchar(10));
```

## 2. mavenプロジェクト作成
先ほどのtableに適当なレコードを挿入する最小限のJavaプロジェクトを実装します。

JDBC、JUnit、maven-assembly-pluginを使用します。

```xml:pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>aaaanwz</groupId>
	<artifactId>test</artifactId>
	<version>0.0.1</version>
	<name>testproject</name>
	<properties>
		<maven.compiler.source>11</maven.compiler.source>
		<maven.compiler.target>11</maven.compiler.target>
	</properties>
	<dependencies>
		<!-- JDBC driver -->
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>8.0.17</version>
		</dependency>
		<!-- JUnit -->
		<dependency>
			<groupId>org.junit.jupiter</groupId>
			<artifactId>junit-jupiter-engine</artifactId>
			<version>5.3.1</version>
			<scope>test</scope>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<!-- JUnit test -->
			<plugin>
				<artifactId>maven-surefire-plugin</artifactId>
				<version>3.0.0-M3</version>
			</plugin>
			<!-- Build runnable jar -->
			<plugin>
				<artifactId>maven-assembly-plugin</artifactId>
				<executions>
					<execution>
						<phase>package</phase>
						<goals>
							<goal>single</goal>
						</goals>
					</execution>
				</executions>
				<configuration>
					<archive>
						<manifest>
							<mainClass>testpackage.Main</mainClass>
						</manifest>
					</archive>
					<descriptorRefs>
						<descriptorRef>jar-with-dependencies</descriptorRef>
					</descriptorRefs>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>
```

1レコード挿入するだけのMainクラスを実装します。  
データベースへの接続情報は環境変数から取得します。

```java:src/main/java/testpackage/Main.java
package testpackage;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;

public class Main {

  public static void main(String[] args) throws ClassNotFoundException, SQLException {
    final String host = System.getenv("DB_HOST");
    final String dbname = System.getenv("DB_NAME");
    execute(host, dbname);
  }

  static void execute(String host, String dbname) throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.cj.jdbc.Driver");
    Connection conn = DriverManager.getConnection(
        "jdbc:mySql://" + host + "/" + dbname + "?useSSL=false", "root", "");
    PreparedStatement stmt = conn.prepareStatement("INSERT INTO user(id,name) VALUES(?, ?)");
    stmt.setInt(1, 1);
    stmt.setString(2, "Yamada");
    stmt.executeUpdate();
  }

}
```

レコードが挿入された事を確認するだけのテストを書きます。

```java:src/test/java/testpackage/MainTest.java
package testpackage;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.fail;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import testpackage.Main;

class MainTest {
  Statement stmt;

  @BeforeEach
  void before() throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.cj.jdbc.Driver");
    Connection conn =
        DriverManager.getConnection("jdbc:mySql://localhost/test?useSSL=false", "root", "");
    stmt = conn.createStatement();
  }

  @AfterEach
  void after() throws SQLException {
    stmt.executeUpdate("TRUNCATE TABLE user");
  }

  @Test
  void test() throws Exception {
    Main.execute("localhost", "test");
    try (ResultSet rs = stmt.executeQuery("SELECT * FROM user WHERE id = 1;")) {
      if (rs.next()) {
        assertEquals("Yamada", rs.getString("name"));
      } else {
        fail();
      }
    }
  }
}

```

## 3. Dockerfile作成

Dockerfileを書きます。テストは `docker build`とは別のステップで行う予定のため、`-DskipTests`を付与してビルド時のテストをスキップします。  
  
イメージサイズ削減のためmaven:3.6でビルド、openjdk11:apline-slimで実行するマルチステージビルドを行います。
maven:3.6では約800MB、openjdk11:alpine-slimでは約300MBとなります。現時点でECRの無料枠は500MBのため、個人開発では大きな差です。

```dockerfile:Dockerfile
FROM maven:3.6 AS build

ADD . /var/tmp/testproject/
WORKDIR /var/tmp/testproject/
RUN mvn -DskipTests package

FROM adoptopenjdk/openjdk11:alpine-slim

COPY --from=build /var/tmp/testproject/target/test-0.0.1-jar-with-dependencies.jar /usr/local/
CMD java -jar /usr/local/test-0.0.1-jar-with-dependencies.jar.jar
```

## 4. k8s yamlファイル作成

3.でビルドされたコンテナをk8sにデプロイするためのyamlを書きます。今回のプログラムは単発でSQLを実行して終了するため、`kind: Job`にしてみます。Docker registryのurlは適宜置換してください。  
`env`でDBへの接続情報を定義します。 `DB_HOST`の値が `mysql`となっているのは、KubernetesのService経由で接続するためです。  


```yaml:k8s-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: test
spec:
  template:
    spec:
      containers:
      - name: test
        image: your-docker-registry-url/testimage:latest
        imagePullPolicy: Always
        env:
        - name: DB_HOST
          value: "mysql"
        - name: DB_NAME
          value: "ekstest"
      restartPolicy: Never
  backoffLimit: 0
```


## 5. circleci configファイル作成

本稿のキモです。CircleCIで自動テスト/ビルドを行うための構成設定を書きます。  

```yml:.circleci/config.yml
version: 2.1
orbs:
  aws-ecr: circleci/aws-ecr@6.1.0
  aws-eks: circleci/aws-eks@0.2.1
  kubernetes: circleci/kubernetes@0.3.0
jobs:
  test: # ①JUnit テストの実行
    docker:
      - image: circleci/openjdk:11
      - image: circleci/mysql:5.7
        environment:
          MYSQL_ALLOW_EMPTY_PASSWORD: yes
          MYSQL_DATABASE: test
        command: [--character-set-server=utf8, --collation-server=utf8_general_ci, --default-storage-engine=innodb]
    steps:
      - checkout
      - run:
          name: Waiting for MySQL to be ready
          command: dockerize -wait tcp://localhost:3306 -timeout 1m
      - run:
          name: Install MySQL CLI; Create table;
          command: sudo apt-get install default-mysql-client && mysql -h 127.0.0.1 -uroot test < schema.sql
      - restore_cache:
          key: circleci-test-{{ checksum "pom.xml" }}
      - run: mvn dependency:go-offline
      - save_cache:
          paths:
            - ~/.m2
          key: circleci-test-{{ checksum "pom.xml" }}
      - run: mvn test
      - store_test_results:
          path: target/surefire-reports
  deploy: # ③EKSへのkubectl apply
    executor: aws-eks/python3
    steps:
      - checkout
      - kubernetes/install
      - aws-eks/update-kubeconfig-with-authenticator:
          cluster-name: test-cluster
          aws-region: "${AWS_REGION}"
      - run:
          command: |
            kubectl apply -f k8s-job.yaml
          name: apply
workflows:
  test-and-deploy:
    jobs: 
      - test: # ①JUnit テストの実行
          filters:
            branches:
              only: develop
      - aws-ecr/build-and-push-image: # ②コンテナのビルドとECRへのpush
          filters:
            branches:
              only: master
          create-repo: true
          repo: "testimage"
          tag: "latest"
      - deploy: # ③EKSへのkubectl apply
          requires:
            - aws-ecr/build-and-push-image
```

### ①JUnitテストの実行
filterによって、`develop`ブランチに変更が加わった際に `mvn test`が実行されるようにしてみました。この際テスト用MySQLインスタンスが立ち上がり、 `test`データベースが準備されます。

公式ドキュメントで詳しく解説されています。　　
- [Language Guide: Java](https://circleci.com/docs/2.0/language-java/)
- [Database Configuration Examples](https://circleci.com/docs/2.0/postgres-config/)

### ②コンテナのビルドとECRへのpush
`master`ブランチに変更が加わった際に、Dockerfileに従ってbuildし、ECRにpushします。  
[Orbクイックスタートガイド](https://circleci.com/orbs/registry/orb/circleci/aws-ecr)で詳しく解説されています。

### ③EKSへのkubectl apply
②が実施された後、4.で定義したJobを実行します。  
[circleci/aws-eks](https://circleci.com/orbs/registry/orb/circleci/aws-eks)に解説がありますが、サンプルコードをよりシンプルに変更しました。  
@0.2.1のドキュメントによるとパラメータ`aws-region`はRequiredとなっていませんが、実際は必須のようです。ECRのpushで環境変数`AWS_REGION`を要求されているので、これをそのまま使いました。


# インフラ設定
ほとんど公式ドキュメントへのリンク集です。

## 1. ECR, EKS環境準備

### 1-1. 基本設定
`test-cluster`を立ち上げます。
- [Getting Started with Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
- [Setting Up with Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/get-set-up-for-amazon-ecr.html)

### 1-2. EKS上にMySQLをデプロイ
Kubernetes公式に[Deploy MySQL](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/#deploy-mysql)というドキュメントが用意されています。  
`kubectl exec -it mysql-0 mysql`コマンドから`ekstest`データベースと`user`テーブルを用意します。

## 2. CircleCI projectの設定

### 2-1. プロジェクトのセットアップ
- [quick-start](https://circleci.com/docs/enterprise/quick-start/)

### 2-2. 環境変数の設定
- [circleci/aws-ecr](https://circleci.com/orbs/registry/orb/circleci/aws-ecr)でRequiredとなっている環境変数をCircleCI Projectに[設定](https://circleci.com/docs/2.0/env-vars/#setting-an-environment-variable-in-a-project)します。   
Roleに要求される権限の詳細は割愛します。

# まとめ
- develop branchにpush  
　mvn testが実行され、テストレポートがCircleCIのTest summaryに表示されます。
- master branchにpush  
  ECRにdocker imageがpushされ、EKSでJobが開始します。EKS上のMySQLに `id:1 name:Yamada`レコードが挿入されます。
