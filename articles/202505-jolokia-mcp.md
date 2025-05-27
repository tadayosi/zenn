---
title: "Jolokia MCP ServerでJavaアプリをLLMから操作する"
emoji: "🌶️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ai", "mcp", "java", "jolokia", "jmx"]
published: false
---

Javaにはもともと[JMX (Java Management Extensions)](https://docs.oracle.com/en/java/javase/21/docs/api/java.management/javax/management/package-summary.html)という、アプリケーション管理用の内部APIが備わっています。JavaベースのほとんどのサーバーやフレームワークがJMXに基づく管理APIを提供しており、システム監視ツールを使ってそのAPIから各種のモニタリングをしていました。

このJMXを使えば、Javaに新しい仕組みやフレームワークを導入せずとも、どんなJavaアプリケーションでもMCPを介してLLMから複雑な操作を行うことができます。

JavaアプリケーションをMCPで操作するには、JMX over HTTPの技術であるJolokiaを使います。

https://github.com/jolokia/jolokia-mcp-server

## Jolokiaとは

[Jolokia](https://jolokia.org/)はJMXをHTTP経由でアクセスするためのライブラリです。Jolokiaは[エージェント](https://jolokia.org/reference/html/manual/agents.html)と[クライアント](https://jolokia.org/reference/html/manual/clients.html)を提供していて、基本的にエージェントをJavaアプリケーションにアタッチすることでJMX over HTTPを有効にします。

エージェントは大きくJVMエージェントとWARエージェントの2種類あり、起動時にJVMにアタッチするならJVMエージェント、WebアプリケーションにJolokiaエンドポイントをデプロイするならWARエージェントを使います。

![Jolokiaアーキテクチャ](https://jolokia.org/reference/html/manual/_images/architecture.png)
*Jolokiaアーキテクチャ [^1]*

[^1]: <https://jolokia.org/reference/html/manual/architecture.html#agent-mode> より。

## Jolokia MCP Server

JolokiaがアタッチされたJVMをMCPサーバー化してくれるのが[Jolokia MCP Server](https://github.com/jolokia/jolokia-mcp-server)です。

まずはこの動画で、JavaアプリをMCP Serverで接続するとどんなことができるのかを見てみてください。動画では、

> check the memory state of the java app
> (Javaアプリのメモリー状態をチェックして)

と指示しています。

https://www.youtube.com/watch?v=Ed0UdlF_erg

このデモはJavaが標準で提供する`java.lang:type=Memory`だけを使ったシンプルな操作ですが、アプリケーションサーバーやフレームワークがより高度なMBean (JMX API)を提供していれば、AIモデルが勝手に利用可能なJMX MBeanの属性や操作をチェックして、タスク達成に必要な操作を考えて実行してくれます。

つまり、もし開発中のJavaアプリケーションを何らかの特定の方法でLLMのプロンプトから操作したい場合、その特定の機能をMBeanとして実装してMBeanサーバーに登録するだけでいいのです。

### 機能・ツール

Jolokia MCP Serverは1インスタンス毎に1つのJVMと接続し、以下の機能をMCPホストに提供します。

- 利用可能なMBeanの一覧を取得
- MBeanの操作一覧を取得
- MBeanの属性一覧を取得
- MBeanの属性への読み書き
- MBeanの操作を実行

MCPツールとしては以下の6つを提供します。

- **listMBeans**
  - JVMから利用可能なMBeanの一覧を取得
  - 出力 (`List<String>`): JVM内のすべてのMBeanオブジェクト名のリスト
- **listMBeanOperations**
  - 指定されたMBeanで利用可能な操作の一覧を取得
  - 入力:
    - `mbean` (`String`): MBean名
  - 出力 (`String`): 指定されたMBeanで利用可能な操作の定義 (JSON形式)
- **listMBeanAttributes**
  - 指定されたMBeanで利用可能な属性の一覧を取得
  - 入力:
    - `mbean` (`String`): MBean名
  - 出力 (`String`): 指定されたMBeanで利用可能な属性の定義 (JSON形式)
- **readMBeanAttribute**
  - 指定されたMBean属性の値を読み取る
  - 入力:
    - `mbean` (`String`): MBean名
    - `attribute` (`String`): 属性名
  - 出力 (`String`): 指定された属性の値の文字列表現、または "null"
- **writeMBeanAttribute**
  - 指定されたMBean属性に値を設定する
  - 入力:
    - `mbean` (`String`): MBean名
    - `attribute` (`String`): 属性名
    - `value` (`Object`): 属性値
  - 出力 (`String`): 指定された属性の以前の値の文字列表現、または "null"
- **executeMBeanOperation**
  - 指定されたMBeanの操作を実行する
  - 入力:
    - `mbean` (`String`): MBean名
    - `operation` (`String`): 操作名
    - `args` (`Object[]`): 引数
  - 出力 (`String`): 操作の戻り値の文字列表現、または "null"

## 使い方

### 1. JavaアプリにJolokiaエージェントを追加

ここでは既存Javaアプリをそのまま使えるように、JVMエージェントを使います。まず、`jolokia-agent-jvm-<version>-javaagent.jar`をダウンロードします。

⬇️ [jolokia-agent-jvm-2.2.9-javaagent.jar](https://repo1.maven.org/maven2/org/jolokia/jolokia-agent-jvm/2.2.9/jolokia-agent-jvm-2.2.9-javaagent.jar)

次にJavaアプリ (`app.jar`) を`-javaagent`を付けて起動します。[^2]

[^2]: Spring Bootの場合は、`spring-boot-starter-actuator`およびJMXが有効になっていることを確認してください。
https://docs.spring.io/spring-boot/reference/actuator/jmx.html

```console
java -javaagent:jolokia-agent-jvm-2.2.9-javaagent.jar -jar app.jar
```

Jolokiaエージェントが正しくアタッチされていれば、最初に以下のようなメッセージが出ます。デフォルトで http://localhost:8778/jolokia/ がJolokiaのエンドポイントになります。

```console
I> No access restrictor found, access to any MBean is allowed
Jolokia: Agent started with URL http://127.0.0.1:8778/jolokia/
```

### 2. Jolokia MCP Serverをインストール

まずJolokia MCP ServerのJARをダウンロードします。

⬇️ [jolokia-mcp-server-0.3.5-runner.jar](https://github.com/jolokia/jolokia-mcp-server/releases/download/v0.3.5/jolokia-mcp-server-0.3.5-runner.jar)

次にMCPホスト (Claude、Cline、Cursorなど) の設定に追加します。Jolokia MCP Server自体は以下のようにして起動できるので、

```console
java -jar jolokia-mcp-server-0.3.5-runner.jar
```

MCPホストの設定にはこのようにします。`<path-to-the-runner-jar>`は適切な絶対パスに置き換えてください。

```json
{
  "mcpServers": {
    "jolokia": {
      "command": "java",
      "args": [
        "-jar",
        "<path-to-the-runner-jar>/jolokia-mcp-server-0.3.5-runner.jar"
      ]
    }
  }
}
```

:::message
Jolokiaエンドポイントをデフォルト (http://localhost:8778/jolokia/) 以外にした場合は、`args`の第3引数にそのエンドポイントURLを指定します。
:::

### 3. MCPホストの起動

MCPホストを起動して、Jolokia MCP Serverがちゃんとインストールされていることを確認します。

Claude Desktopの場合、以下のようにサーバーが認識されツールが読み込まれていれば成功です。

![Claude Desktop](/images/202505-jolokia-mcp/claude-jolokia.png)
*Claude Desktopの場合*

試しに、

> メモリの使用状況をチェック

などとプロンプトを入力してみましょう。上手く行ったら、次は色々なプロンプトを試してみてください。

## まとめ

JavaアプリケーションをLLMから操作できるJolokia MCP Serverを紹介しました。

JolokiaエージェントをJavaアプリケーションにアタッチすることで既存のJMX MBeanをMCPツールに変え、LLMを通して直接Javaアプリケーションと対話、操作できるようになります。

MCP、LLMを使った新しいJavaアプリケーション管理、自動化の可能性がここにあります。エンタープライズJavaにおけるLLM活用の1つの方向性と言えるでしょう。

Jolokia MCP Serverは今後もさらに改良していく予定です。例えば、

- [JolokiaエージェントとMCP Serverの一体化](https://github.com/jolokia/jolokia-mcp-server/issues/12)
- [ストリーミングHTTPのサポート](https://github.com/jolokia/jolokia-mcp-server/issues/13)
- [定型プロンプトの提供](https://github.com/jolokia/jolokia-mcp-server/issues/14)

などが検討されています。

ぜひJolokia MCP Serverを試してみてください。
