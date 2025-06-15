---
title: "Javaはスクリプト言語だ — JBangが変えるJava開発の未来"
emoji: "☕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["jbang", "java", "script", "scripting"]
published: false
---

[JJUG CCC 2025 Spring](https://ccc2025spring.java-users.jp/)にて「[Javaはスクリプト言語だ — JBangが変えるJava開発の未来](https://sessionize.com/api/v2/s2ztutnz/view/Sessions#sz-session-872648)」という発表をさせていただきました。

当日の発表スライドとデモコードはこちらに公開しています。

https://github.com/tadayosi/jjug2025-jbang

忘れないうちに、話した内容をここに記事にしておこうと思います。

## JBang

JBangは、Javaをスクリプトのように直接実行できるツールです。

https://www.jbang.dev/

### なぜJBang？

Javaで開発をしていると、ちょっとしたライブラリやAPIの挙動を検証したくなるときがあります。

開発中のプロジェクトにそうした検証コードを直接書いたりするのは嫌なので、通常は新たに検証用のプロジェクトを作ることになります。IDEやコマンドラインからJavaのプロジェクトを作り、`pom.xml`や`build.gradle`をいじって必要なプロジェクト設定と依存関係を追加し、やっと検証コードを書けるようになります。

Javaはビルドや実行は非常に速く快適ですが、このプロジェクトを作る部分が地味にフットワークが重くなります。JBangを使うと、コマンドラインからスクリプトを1つ生成するだけで、すぐにVimやEmacsなどのお好みのエディターでコードを書き始めることができます。

そしてJBangを使い込んでくると、1プロジェクト分の情報量を1スクリプトに凝縮して詰め込めるようになります。つまり、1スクリプト ＝ 1プロジェクトの感覚でJavaコードを書くことができます。

同時に重要なのは、JBangはスクリプト的な手軽さはあっても、実行ランタイムはあくまで本番環境と同じJVMだということです。手軽に開発、検証を始められながら、ビルドして得られる実行コードは本番環境と同じ速度と堅牢性を備えています。

さらに、IDEとの連携や、スクリプトの配布方式などのエコシステムも充実しています。スクリプト的に書きなぐっていたところから、その先に発展させるパスも用意されています。そのため、マイクロサービス的な開発にも向いているでしょう。

このようにJBangを使い続けていると、単なる便利ツールでなく次世代のJava開発環境のように見えてきます。スモールスタートで始めてそこからフルスペックの開発に発展させる、あるいはサンプルコード、検証コードなどのチームで共有するナレッジベースを効果的に構築する、そういったことを実現する新しいプラットフォームです。

## インストール

JBangのダウンロードページに様々なインストール方法が示されています。

https://www.jbang.dev/download/

:::message
お勧めの方法は[sdkman](https://sdkman.io/)です。sdkmanはJava専用のパッケージマネージャーで、JBangだけでなく各社のJDKディストリビューションから、Maven、Gradle、Spring Boot/Quarkus CLIなど、他の様々なJavaベースのコマンドツールまでまとめてインストールして管理できます。
:::

### curl

```console
curl -Ls https://sh.jbang.dev | bash -s - app setup
```

### sdkman

```console
sdk install jbang
```

### homebrew

```console
brew install jbangdev/tap/jbang
```

## はじめる

### init

`init`コマンドを使ってスクリプトファイルを作ります。

```console
jbang init hello.java
```

https://github.com/tadayosi/jjug2025-jbang/blob/main/1-init/hello.java

スクラッチから書き始めてもいいですが、`init`を使うとシェルスクリプトのように実行するための作法が予め埋め込まれたスクリプトが生成されます。

### run

`init`で生成されたスクリプトは、シェルスクリプトのように直接実行可能です。

```console
./hello.java
```

普通に`jbang`コマンドから実行することもできます。

```console
jbang hello.java
```

他にも[色々と面白い実行方法](https://www.jbang.dev/documentation/guide/latest/usage.html)がありますが、たとえばこのようにGistにスクリプトをアップロードしてそれをURLから実行することもできます。

```console
jbang https://gist.github.com/tadayosi/43dc74403c7acce5384c23129fbeb918
```

### build

JARにビルドもできます。ビルドしておけば、JBangがない環境でもJVMだけで実行できるようになります。

```console
jbang build --build-dir <path> hello.java
```

### native

GraalVMがインストールされていれば、ネイティブビルドも可能です。組み込み環境など、実行速度やフットプリントが気になるようなユースケースで有効です。

```console
jbang build --build-dir <path> --native hello.java
```

### ファイルヘッダーの秘密

`init`で生成されたスクリプトを見ると、次のファイルヘッダーが定義されています。

https://github.com/tadayosi/jjug2025-jbang/blob/main/1-init/hello.java#L1

これはシェルスクリプトのシバン（shebang） `#!` をマネするハックです。リンク先の記事、Goクックブックでこのテクニックが説明されています。

https://golangcookbook.com/chapters/running/shebang/

仕組みは次の通りです。まず、シェルでスクリプトとして実行されて先頭行が読まれたときに、スラッシュ（`///`）はパスの区切り文字として扱われるため、絶対パスで指定された`env`を呼び出す命令として実行されます。つまり、

```bash
/usr/bin/env jbang "hello.java" ""; exit $?
```

と同等です。次に、再帰的に`jbang`コマンドが呼び出されたときには、先頭行のスラッシュ（`//`）はJavaではコメント行なので、無視されてそこから先は有効なJavaソースとして実行されます。

これがJBangスクリプトのファイルヘッダーの秘密です。

## 依存のインポート

Javaの強みは、豊富なライブラリーによるエコシステムです。JBangでも、当然その強みを最大限活用できます。

ファイルのヘッダー部分に`DEPS`というコメントタグを使うことで、依存ライブラリーをインポートできます。

https://github.com/tadayosi/jjug2025-jbang/blob/main/2-import/log.java#L3-L4

指定の仕方は、いわゆるMavenのGAV (Group ID, Artifact ID, Version) 形式のコーディネートで指定します。Gradleを使ってる方はおなじみの形式です。

### BOMも使える

JBangでは、依存ライブラリーのバージョンの後に`@pom`を付けることでBOMを読み込めます。

https://github.com/tadayosi/jjug2025-jbang/blob/main/2-import/bom.java#L3-L5

この例では、SLF4JのBOMを読み込んでいます。それ以降、`slf4j-api`や`slf4j-simple`といったサブコンポーネントはバージョンを指定せずにインポートできます。

:::message
**BOM (Bill of Materials)** とは製造業における部品表のことで、ソフトウェアにおいてはライブラリやフレームワークが、そのサブコンポーネントや依存ライブラリーのバージョン一覧を定義したものです。

MavenやGradleはBOMをサポートしていて、BOMを読み込むことでそこに定義されたサブコンポーネントや依存ライブラリーについてはわざわざバージョンを指定する必要がなくなります。
:::

### Maven Central Search (`mcs`コマンド)

便利コマンドを1つ紹介します。

Maven Central Search (`mcs`) というコマンドは、Mavenコーディネートの一部をキーに検索して、Maven Centralリポジトリーからその最新のバージョン一覧を取得してきてくれます。

https://github.com/mthmulders/mcs

```console
$ mcs org.slf4j:slf4j-api
...

  Coordinates                        Last updated
  ===========                        ============
  org.slf4j:slf4j-api:2.0.17         26 Feb 2025 at 01:43 (JST)
  org.slf4j:slf4j-api:2.0.16         10 Aug 2024 at 18:15 (JST)
  org.slf4j:slf4j-api:2.0.15         08 Aug 2024 at 21:59 (JST)
  ...
```

JBangとは直接関係のないツールですが、JBangで依存ライブラリーを調べるときに最適です。

### リポジトリー

Mavenリポジトリーは、デフォルトではCentralを見に行きますが、社内のオフライン環境でイントラネット内のリポジトリーだけを参照したい場合や、製品ベンダーの特定のリポジトリーを使う必要がある場合などに、`REPOS`コメントタグで指定できます。

```java
//REPOS central,jitpack,myrepo=https://myrepo.local/maven
```

デフォルトで定義されてる`central`、`google`、`jitpack`の他に、自分で`myrepo=https://myrepo.local/maven`のようにリポジトリーIDとURLのペアで指定することもできます。

| 名前 | 説明 |
| ---- | ---- |
| `central` | <https://repo1.maven.org/maven2/> |
| `google` | <https://maven.google.com/> |
| `jitpack` | <https://jitpack.io/> |

:::message
[JitPack](https://jitpack.io/)とは、GitHub上に公開したあらゆるJavaプロジェクトを自動でビルドして専用のリポジトリーから公開してくれるWebサービスです。自分のJavaプロジェクトをわざわざMaven Centralに上げなくても、JitPackを通して通常の依存ライブラリーのように扱えます。
:::

## その他のタグ

そのほかJBangでサポートしているタグはたくさんありますが、独断で重要そうなものをピックアップすると以下の通りです。

```java
//JAVA 21+

//COMPILE_OPTIONS --enable-preview --verbose

//RUNTIME_OPTIONS --add-opens java.base/java.net=ALL-UNNAMED

//NATIVE_OPTIONS -O1 -g

//PREVIEW

//MODULE <module-name>

//MAIN <main-class-name>

//MANIFEST Built-By=JBang

...
```

:::message
とくに`JAVA`タグは、使用するJavaバージョンを明示的に指定できます。`+`（プラス）を付ければJavaの最低バージョンを指定できます。スクリプトで最新のJava構文などを試したいときに役立ちます。
:::

## マルチファイル

依存ライブラリー読み込みの他に不可欠なのは、スクリプトから別のソースを読み込むことです。ロギングやSpring Boot、Quarkusのように、特別なリソースファイルを設定ファイルとして読み込む必要のあるフレームワークもあります。

ファイルを読み込むには、Javaソースなら`SOURCES`、リソースファイルなら`FILES`コメントタグを使います。

```java
//SOURCES GreetingService.java
//SOURCES model/Person.java
```

```java
//FILES application.properties
//FILES META-INF/resources/index.html=index.html
```

リソースファイルについては、`META-INF/resources/index.html=index.html`のように`=`を使って右側のローカルファイル (`index.html`) を左側のパス (`META-INF/resources/index.html`) として読み込ませることも可能です。Jakarta EE系のように、`META-INF`以下にリソースファイルを置く必要のある場合に有効です。

## IDEとの連携

依存ライブラリーやマルチファイルを使ってスクリプトを書いていると、IDEのコード補完などのサポートが欲しくなります。`jbang edit`コマンドを使うと、JBangから指定のIDEを立ち上げることができます。

```console
jbang edit greet.java
```

`--sandbox`オプションを付けると、新たなサンドボックスプロジェクト(ディレクトリ)を作ってそこでIDEを開きます。スクリプトとそこから[参照されているJavaソースおよびリソースファイル](#マルチファイル)は、サンドボックスプロジェクトにシンボリックリンクとしてコピーされます。そのため、そこで編集した内容は元のソースコードに直接反映されます。

```console
jbang edit --sandbox greet.java
```

さらに`--no-open`オプションを付けると、IDEを開かず、サンドボックスプロジェクトのパスだけを返します。これをコマンドラインからIDEに渡して立ち上げることで、任意のIDEを使ってサンドボックスプロジェクトの編集を行えます。

```console
$ jbang edit --sandbox --no-open greet.java
[jbang] Creating sandbox for script editing greet.java
/Users/tadayosi/.jbang/cache/projects/greet.java_jbang_6df7e8896068e72b39f45fca68fbb4c219e412c50af646bdfbfaa1472c26e429/greet
$ idea `jbang edit --sandbox --no-open greet.java`
```

### IDEプラグイン

IDEのJavaサポートに加え、JBangエクステンション/プラグインをインストールすることで、[`DEPS`](#依存のインポート)から依存ライブラリーをインポート(`Synchronize JBang`)したり、Javaエディターから直接JBangスクリプトを実行(`Run JBang`)、デバッグ実行(`Debug JBang`)したりできるようになります。

以下がVS CodeとIntelliJのエクステンション/プラグインです。

- [VS Code](https://marketplace.visualstudio.com/items?itemName=jbangdev.jbang-vscode)
- [IntelliJ](https://plugins.jetbrains.com/plugin/18257-jbang)

### エクスポート

さらに、JBangスクリプトでなく通常のJavaプロジェクトとして開発を続けたくなったら、MavenまたはGradleプロジェクトにエクスポートできます。

```console
jbang export maven app.java
jbang export gradle app.java
```

指定したスクリプトの他、そこから[参照されているJavaソースおよびリソースファイル](#マルチファイル)がプロジェクトのソースコードとしてエクスポートされます。また、スクリプトに[指定した依存ライブラリー](#依存のインポート)が`pom.xml`または`build.gradle`に追加されます。

## 実践編

ここまでがJBangの基本的な使い方でした。

実践編では、実際に私がこれまでJBangを使ってやってきたことを元に、JBangでどんなことまでできるのかを紹介します。

1. [REST](#rest)
2. [gRPC](#grpc)
3. [AI/LLM](#ai%2Fllm)

### REST

ちょっとした検証や、システムの統合テスト用にモックのRESTサービスが必要になったときなどに、JBangを使って簡単にRESTサービスを実装できます。

#### Quarkus

Quarkusを使ったRESTサービスのスクリプトです。Quarkusのサポートにより、`main`メソッドがなくてもこれだけでJBangからRESTサービスを起動できます。

https://github.com/tadayosi/jjug2025-jbang/blob/main/4-usages/1-rest/quarkus_rest.java

#### Spring Boot

Spring BootでもRESTサービスを実装できます。Spring Bootでは`main`メソッドが必要です。またコンポーネントスキャンの仕様上、デフォルトパッケージ以外のパッケージ名を使用する必要があります。

https://github.com/tadayosi/jjug2025-jbang/blob/main/4-usages/1-rest/rest/sb_rest.java

### gRPC

gRPCの通信をテストするのにもJBangが使えます。次の`.proto`ファイルから、gRPCで通信するクライアントとサーバーを実装してみます。

https://github.com/tadayosi/jjug2025-jbang/blob/main/4-usages/2-grpc/hello.proto

まず、`protoc`コマンドを使って`.proto`ファイルからProtocol BuffersのモデルクラスとgRPCスタブを生成する必要があります。

```console
# モデルクラスの生成
protoc --java_out=. ./hello.proto
```

```console
# gRPC Javaスタブの生成
protoc \
  --plugin=protoc-gen-grpc-java=./protoc-gen-grpc-java-${GRPC_VERSION}-${OS}-${ARCH}.exe \
  --grpc-java_out=. \
  ./hello.proto
```

#### サーバー

生成された以下のJavaソースファイルを読み込んで、`HelloService`を実装したgRPCサーバーです。

- hello/Hello.java
- hello/HelloServiceGrpc.java

https://github.com/tadayosi/jjug2025-jbang/blob/main/4-usages/2-grpc/server.java

#### クライアント

同じく生成されたJavaソースファイルを読み込んで、gRPCクライアントを実装します。

https://github.com/tadayosi/jjug2025-jbang/blob/main/4-usages/2-grpc/client.java

### AI/LLM

AIやLLMのテストや学習も、JBangを使って行えます。

#### DJL

Javaの深層学習フレームワークである[DJL](https://djl.ai/)を使ったサンプルスクリプトです。ここではAIモデルのMLPを使って、手書き数字の画像認識を行っています。

https://github.com/tadayosi/jjug2025-jbang/blob/main/4-usages/3-ai/djl.java

#### LangChain4j

最後に、[LangChain4j](https://docs.langchain4j.dev/)と[Ollama](https://ollama.com/)を使ったローカルで軽量LLMを動かすサンプルスクリプトです。Llama 3.2とLangChain4jのAIサービスの機能を使って、LLMにコーヒーの作り方を聞いています。

https://github.com/tadayosi/jjug2025-jbang/blob/main/4-usages/3-ai/llm.java

## CLI開発

最後に、JBangのエコシステムとその機能について紹介します。それにはまず、CLI開発について触れる必要があります。

シェルスクリプトでちょっとしたCLI、コマンドラインツールを作るのは定番ですが、同様にJBangでCLIツールを作ることができます。

```console
jbang init --template=cli cli.java
```

次のようなスクリプトの雛形が生成されます。

https://github.com/tadayosi/jjug2025-jbang/blob/main/5-cli/cli.java

### picocli

JBangのCLIテンプレートで使われているのは、picocliというCLIフレームワークです。

https://picocli.info/

### JBangベースのCLI

JBang + picocliを使って、実際にCLIを提供しているJavaのフレームワークやアプリケーションがあります。

- [Camel](https://camel.apache.org/manual/camel-jbang.html) — システム間インテグレーションのフレームワーク
- [Hawtio](https://hawt.io/docs/get-started.html) — JMXベースのWeb管理コンソール
- [Citrus](https://citrusframework.org/samples/jbang/) — BDDインテグレーションテストフレームワーク

### カタログ

さらにJBangにはカタログという、スクリプト配布のための仕組みが備わっています。先ほどのCLI開発とこのカタログを組み合わせることで、JBangによるエコシステム構築の土台が出来上がります。

カタログとは`jbang-catalog.json`という名前のJSONファイルです。これをGitHubリポジトリー`https://github.com/user/repo`に置くと、

```console
jbang my-cli@user/repo
```

のようにリポジトリーから直接スクリプトを呼び出すことができます。

とくにリポジトリー名を`jbang-catalog`にすると、

```console
jbang my-cli@user
```

のように`コマンド名@ユーザー名`だけでスクリプトを実行できます。

### 例

実際にApache CamelのJBangカタログを見てみましょう。

https://github.com/apache/camel/blob/main/jbang-catalog.json

`script-ref`で参照されているスクリプトは次の通りです。シンプルなスクリプトが、依存として特定バージョンのCamelライブラリーを読み込むことでCLIを起動します。

https://github.com/apache/camel/blob/main/dsl/camel-jbang/camel-jbang-main/dist/CamelJBang.java

このCamel CLIは次のコマンドで起動できます。

```console
jbang camel@apache/camel
```

### アプリとしてインストール

それに加えてアプリという機能もあります。カタログのスクリプトを、コマンドとしてローカルにインストールできる機能です。

Camel CLIの例を使うと、次のようにするだけで`camel`コマンドをローカルにインストールできます。

```console
jbang app install camel@apache/camel

camel --version
```

ここまでの機能によって、JBangのコード配布のエコシステムとしての仕組みが完成します。

### 公式カタログ

最後に、主要なベンダーでもJBangカタログを公開する流れが少しずつ増えています。

```console
$ jbang catalog list
alibaba
   https://github.com/alibaba/jbang-catalog/blob/HEAD/jbang-catalog.json
apache/camel
   Run Apache Camel routes easily
   https://github.com/apache/camel/blob/HEAD/jbang-catalog.json
jbangdev
   JBang's own catalog of small utilities
   https://github.com/jbangdev/jbang-catalog/blob/HEAD/jbang-catalog.json
jbanghub [importing]
   JBangHub - Unleashing the Java community
   https://raw.githubusercontent.com/jbanghub/jbang-catalog/main/jbang-catalog.json
jbanghub/apple
   Apple cli's
   https://github.com/jbanghub/apple/blob/HEAD/jbang-catalog.json
jbanghub/eclipse
   Eclipse Foundation developed tools
   https://github.com/jbanghub/eclipse/blob/HEAD/jbang-catalog.json
jbanghub/h2
   H2 Database Engine
   https://github.com/jbanghub/h2/blob/HEAD/jbang-catalog.json
jbanghub/jvm-profiling-tools
   A collection of JVM profiling tools
   https://github.com/jvm-profiling-tools/ap-loader/blob/HEAD/jbang-catalog.json
jbanghub/sqlline
   Shell for issuing SQL to relational databases via JDBC
   https://github.com/jbanghub/sqlline/blob/HEAD/jbang-catalog.json
jreleaser
   Release projects quickly and easily with JReleaser
   https://github.com/jreleaser/jbang-catalog/blob/HEAD/jbang-catalog.json
jupyter-java
   Make it easy to use java from jupyter notebooks
   https://github.com/jupyter-java/jbang-catalog/blob/HEAD/jbang-catalog.json
jvm-profiling-tools/ap-loader
   Sampling CPU and HEAP profiler for Java featuring AsyncGetCallTrace + perf_events via JBang
   https://github.com/jvm-profiling-tools/ap-loader/blob/HEAD/jbang-catalog.json
maveniverse
   The missing pieces from Apache Maven universe
   https://github.com/maveniverse/jbang-catalog/blob/HEAD/jbang-catalog.json
microsoft
   Catalogue for JBang Artifacts
   https://github.com/microsoft/jbang-catalog/blob/HEAD/jbang-catalog.json
oracle/graalpython
   A Python 3 implementation built on GraalVM
   https://github.com/oracle/graalpython/blob/HEAD/jbang-catalog.json
quarkusio
   Subsonic and Subatomic Quarkus cli and plugins
   https://github.com/quarkusio/jbang-catalog/blob/HEAD/jbang-catalog.json
```

- [Alibaba](https://github.com/alibaba/jbang-catalog)
- [Apple](https://github.com/jbanghub/apple)
- [Microsoft](https://github.com/microsoft/jbang-catalog)
- [Oracle (GraalPy)](https://github.com/oracle/graalpython)

日本からも、JBangカタログでJavaのコードを配布する動きが広まってくれるといいなと思います。

## まとめ

これからのJava開発でJBangを使ってみよう、と思ってくれる方が少しでも増えたら幸いです。

とりあえずJBangをインストールして触ってみるだけでも楽しいので、ぜひ試してみてください。
