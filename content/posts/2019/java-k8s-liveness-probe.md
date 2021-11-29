---
title: "Kubernetes Liveness ProbeでJavaプロセスを監視する"
date: 2019-07-02
draft: false
categories:
- java
- kubernetes
---

Javaプロセスを一定時間毎にチェックし、ハングしていればPodを再起動する仕組みの備忘録です。  
Kubernetes LivenessProbeに関する詳細は[こちら](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-tcp-liveness-probe)をご参照ください。

# Java実装
### 監視対象クラス
テスト用に、インスタンスが生成されてから10秒後に `isAlive() == false`になるように実装します。

```java:SomeResource.java
public class SomeResource {
  final long createdTime;

  public SomeResource() {
    this.createdTime = System.currentTimeMillis();
  }
  public boolean isAlive() {
    return System.currentTimeMillis() - createdTime < 10000;
  }
}
```

### 監視用エンドポイント  

`SomeResource#isAlive() == true`の時はレスポンスコード `200`, `false`の時は `500`を返すように実装します。

```java:HeartBeat.java
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.net.InetSocketAddress;

public class HeartBeat {
  private final SomeResource target;
  private HttpServer httpServer;
  private final int port;

  public HeartBeat(SomeResource target, int port) {
    this.target = target;
    this.port = port;
  }

  void start() throws IOException {
    httpServer =
        HttpServer.create(new InetSocketAddress(port), 0);
    httpServer.createContext("/heartbeat", httpExchange -> {
      int responseCode;
      if (target.isAlive()) {
        responseCode = 200;
      } else {
        responseCode = 500;
      }
      byte[] responseBody = String.valueOf(target.isAlive()).getBytes(StandardCharsets.UTF_8);
      httpExchange.getResponseHeaders().add("Content-Type", "text/plain; charset=UTF-8");
      httpExchange.sendResponseHeaders(responseCode, responseBody.length);
      httpExchange.getResponseBody().write(responseBody);
      httpExchange.close();
    });
    httpServer.start();
  }

  void stop() {
    httpServer.stop(0);
  }
}
```

### Mainクラス
```java:Main.java
import java.util.concurrent.CountDownLatch;

public class Main {

  public static void main(String[] args) {
    final CountDownLatch latch = new CountDownLatch(1);
    SomeResource resource = new SomeResource();
    HeartBeat heartbeat = new HeartBeat(resource, 1234);// ポート1234番を使用

    Runtime.getRuntime().addShutdownHook(new Thread() {
      @Override
      public void run() {
        heartbeat.stop();
        latch.countDown();
      }
    });

    try {
      heartbeat.start();
      latch.await();
    } catch (Exception e) {
      e.printStackTrace();
      System.exit(-1);
    }
  }
}
```

# curlでエンドポイントをテストする
### SomeClassインスタンス生成直後

```console:console
$ curl --dump-header - http://localhost:1234/heartbeat
HTTP/1.1 200 OK
Date: Tue, 02 Jul 2019 07:24:20 GMT
Content-type: text/plain; charset=UTF-8
Content-length: 4

true
```

### 10秒後

```console:console
$ curl --dump-header - http://localhost:1234/heartbeat
HTTP/1.1 500 Internal Server Error
Date: Tue, 02 Jul 2019 07:24:30 GMT
Content-type: text/plain; charset=UTF-8
Content-length: 5

false
```

# Kubernetesにデプロイ
コンテナの作成、レジストリへのpush手順は省略します。  

```yaml:deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-resource-watch-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: java-resource-watch-test
  template:
    metadata:
      labels:
        app: java-resource-watch-test
    spec:
      containers:
      - name: java-resource-watch-test
        image: some_registry/test:latest
        livenessProbe:
          httpGet:
            path: /heartbeat
            port: 8080
            httpHeaders:
            - name: Custom-Header
              value: HealthCheck
          initialDelaySeconds: 1
          periodSeconds: 1
```

`kubectl apply -f ./deployment.yaml`

SomeResourceの寿命が切れると同時にPodが終了し、再起動される事が確認できるかと思います。
