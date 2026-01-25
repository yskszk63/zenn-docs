---
title: "OkHttpをナマケモノにするjavaagentを書いた"
emoji: "🦥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Java"]
published: true
---

# OkHttpをナマケモノにするjavaagentを書いた

## はじめに

API クライアントなどを実装するにあたり、外部サービスの HTTP 429 Too Many Requests をハンドリングしたい場面に出くわすことがあると思います。
動作確認するにあたり、開発中は外部サービスに負荷をかけたくないので、 Too Many Requests が返るようなアクセスのしかたは避けたいですね。
そのような時は、普通はリバースプロキシを実装して、フレーキーに HTTP 429 Too Many Requests を返してあげてしまえば良いかと思います。
ただ、現実には、そのような構成にできない場面ってあったりもします。たとえば、このプロキシを通してインターネットにアクセスしてね。など。

このような状況が実際にあり、それをどうにかしたく [flaky-okhttp-agent](https://github.com/yskszk63/flaky-okhttp-agent) を作りました。
OkHttpClient のレスポンスを HTTP 429 Too Many Requests が返ってきているように見せる作りになっています。
使い方はリポジトリをご参照下さい。

## Javaプログラミング言語エージェントとは

JVM の仕組みで、 Java プログラムの起動時または動的にエージェントをアッタッチさせてバイトコード書き替えをするための仕組みです。
バイトコード書き替えにより、トレーシングやメトリックスの収集、デバッグのための情報収集などの仕組みをソースコードの変更無しに組み込むことができます。

たとえば起動時にこの仕組みを使いたい場合は、下記シグニチャのメソッドを用意し、

```java
 public static void agentmain(String agentArgs, Instrumentation inst) 
```

JAR のマニフェストに `Premain-Class` で先のメソッドを実装したクラスを指定します。

実際に `java` コマンドでその JAR を指定して起動するには、

```bash
java -javaagent:path/to/agent.jar ...
```

のように指定することで機能させることができます。

詳しくは下記をご確認ください。
https://docs.oracle.com/javase/jp/25/docs/api/java.instrument/java/lang/instrument/package-summary.html

なお、バイトコードを手で書いて、、というのはさすがに厳しいのでライブラリが欲しくなってきます。
[Byte Buddy](https://bytebuddy.net/)
が比較てき容易に実装ができて良いように思います。

上記ドキュメントから引用しますが、実装のしかたは下記のようなイメージです。

```java
    class ToStringAgent {
      public static void premain(String arguments, Instrumentation instrumentation) {
        new AgentBuilder.Default()
            .type(isAnnotatedWith(ToString.class))
            .transform(new AgentBuilder.Transformer() {
          @Override
          public DynamicType.Builder transform(
                DynamicType.Builder builder,
                TypeDescription td, ClassLoader cl, JavaModule jm, ProtectionDomain pd) {
            return builder.method(named("toString"))
                          .intercept(FixedValue.value("transformed"));
          }
        }).installOn(instrumentation);
      }
    }
```

→ `@ToString` アノテーションで注釈されたクラスの `toString` メソッドを常に `transformed` で返す

## これなら OkHttpClient.Builder に Interceptor を差し込めそうだね

エントリーポイントとなるメソッドには、 [java.lang.instrument.Instrumentation](https://docs.oracle.com/javase/jp/25/docs/api/java.instrument/java/lang/instrument/Instrumentation.html) が渡され、
このインタフェースにはバイトコード書き替えをするためのメソッドが公開されています。

`OkHttpClient.Builder` のコンストラクタの後に Interceptor を追加できればレスポンスの詐称ができますね。

## 苦労した点

`Advice` のシグニチャに `this` として受け取るオブジェクトのクラスを使うと、 Advice するクラスを読むために改変対象のクラスを読み込む必要がでてきて循環してしまいます。
→ 開発中に `java.lang.LinkageError: loader (instance of  sun/misc/Launcher$AppClassLoader): attempted  duplicate class definition for name: "okhttp3/OkHttpClient$Builder"` のようなエラーが発生した

改変対象のクラスがメソッドのシグニチャに出てきてクラスロードが改変前に発生してしまうことを避けるため、引数としては `Object` で受けとり、内部であらためてキャストすることで回避しました。
