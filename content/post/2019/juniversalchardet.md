---
title: "Javaで文字コードを推測する"
date: 2019-06-27
draft: false
categories:
- java
---

[juniversalchardet](https://code.google.com/archive/p/juniversalchardet/)を使用して、

- ファイルの文字コードを推測・デコード・コンソールへの表示を行う
- URLエンコードされた文字列をデコードする  

の2つのサンプルプログラムを作成してみます。  

juniversalchardetとはMozillaによって提供されているライブラリで、バイト列のパターンの出現頻度をもとに文字コードを推測する機能を提供します。現在日本語ではISO-2022-JP, SHIFT-JIS, EUC-JPに対応しています。

## 開発環境
- OpenJDK 11
- Maven 3.6

## 下準備
以下をmaven dependenciesに追加します

`pom.xml`

```xml
<dependency>
    <groupId>com.googlecode.juniversalchardet</groupId>
    <artifactId>juniversalchardet</artifactId>
    <version>1.0.3</version>
</dependency>
```

## サンプル1. ファイル読み込み
### Detectorクラス
今回は汎用性のためにInputStreamを引数としてみます。
引数に渡されたInputStreamインスタンスはオフセットが進んでしまう事に注意が必要です。
UniversalDetectorは入力データが全てシングルバイト文字の場合は文字コード判定結果がnullとなります。今回はそのような場合は環境デフォルト値を返すようにしました。

`Detector.java`

```java
import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.Charset;
import org.mozilla.universalchardet.UniversalDetector;

public class Detector {
  public static Charset getCharsetName(InputStream is) throws IOException {
    //4kbのメモリバッファを確保する
    byte[] buf = new byte[4096];
    UniversalDetector detector = new UniversalDetector(null);

    //文字コードの推測結果が得られるまでInputStreamを読み進める
    int nread;
    while ((nread = is.read(buf)) > 0 && !detector.isDone()) {
      detector.handleData(buf, 0, nread);
    }
    
    //推測結果を取得する
    detector.dataEnd();
    final String detectedCharset = detector.getDetectedCharset();
    
    detector.reset();

    if (detectedCharset != null) {
      return Charset.forName(detectedCharset);
    }
    //文字コードを取得できなかった場合、環境のデフォルトを使用する
    return Charset.forName(System.getProperty("file.encoding"));
  }
}
```

### Mainクラス

ファイルの文字コードを判別し、コンソールに出力します。  
FileInputStreamはmark/resetをサポートしていないため、文字コード判別とコンソール出力で別のインスタンスを生成します。

`Main.java`

```java
import java.io.BufferedReader;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.nio.charset.Charset;

public class Main {
  public static void main(String[] args) throws IOException {
    final String path = "./test.txt";

    Charset cs;
    try (FileInputStream fis = new FileInputStream(path)) {
      cs = Detector.getCharsetName(fis);
      System.out.println("charset:" + cs);
    }

    try (BufferedReader br =new BufferedReader(new InputStreamReader(new FileInputStream(path), cs))) {
      br.lines().forEach(s -> System.out.println(s));
    }
  }
}
```

### 実行例
```text:実行結果
charset:SHIFT-JIS
あいうえお
```

## サンプル2. URLのデコード

追加でApache commons codecを使用します。

`pom.xml`

```xml
<dependency>
    <groupId>commons-codec</groupId>
    <artifactId>commons-codec</artifactId>
    <version>1.12</version>
</dependency>
```

### Detectorクラス

`Detector.java`

```java
import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.Charset;
import org.mozilla.universalchardet.UniversalDetector;

public class Detector {
  public static Charset getCharsetName(byte[] bytes) {
    UniversalDetector detector = new UniversalDetector(null);
    //入力文字列が短すぎると推測ができないため、入力を繰り返す
    while (!detector.isDone()) {
      detector.handleData(bytes, 0, bytes.length);
      detector.dataEnd();
    }
    final String charsetName = detector.getDetectedCharset();
    detector.reset();
    if (charsetName != null) {
      return Charset.forName(charsetName);
    }
    return Charset.forName(System.getProperty("file.encoding"));
  }
}
```

### Mainクラス

`Main.java`

```java
import java.io.UnsupportedEncodingException;
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;
import org.apache.commons.codec.DecoderException;
import org.apache.commons.codec.net.URLCodec;

public class Main {
  public static void main(String[] args) throws DecoderException, UnsupportedEncodingException {
    final String str= "%82%a0%82%a2%82%a4%82%a6%82%a8";
    //URLエンコード文字列をパースし、バイト配列にする
    byte[] bytes = new URLCodec()
        .decode(str, StandardCharsets.ISO_8859_1.name())
        .getBytes(StandardCharsets.ISO_8859_1.name());
    
    Charset cs = Detector.getCharsetName(bytes);
    System.out.println("charset:"+cs);

    //バイト配列を検出したcharsetを用いて文字列にする
    final String s = new String(bytes,cs);
    System.out.println(s);
  }
}
```

### 実行例

```text:実行結果
charset:SHIFT-JIS
あいうえお
```
