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

Sources

```java
//SOURCES GreetingService.java
//SOURCES model/Person.java
```

Resources

```java
//FILES application.properties
//FILES META-INF/resources/index.html=index.html
```

依存ライブラリを設定する以外で、やりたくなるのはスクリプトから別のソースを読み込むことですね。
ロギングとかSpring Boot、Quarkusみたいに、リソースファイルからの設定が必要なフレームワークもあります。

ファイルを読み込むには、Javaソースなら`SOURCES`、リソースファイルなら`FILES`コメントタグを使います。

リソースファイルについては、ここにあるように`=`（イコール）を使って別のパスにロードさせることも可能です。
`META-INF`系のファイルを設定するときに有効です。

## IDEとの連携

```console
jbang edit greet.java
jbang edit --sandbox greet.java
jbang edit --sandbox --no-open greet.java
```

たぶんマルチファイルとか使いだしてスクリプト書いてるときは、もう単なるちょっとした検証とかでなく、がっつりコードを書きたくなってるところだと思うんですよ。

そうでなくとも、ちょっとしたコードでも色々なライブラリと戯れてるときはIDEの補完使って書きたくなるときありますよね。

そのときは、サクッとIDEを開いちゃいましょう。
今のところVS Code、IntelliJ、Eclipseくらいですが、しっかりサポートされています。

### IDEプラグイン

- [VS Code](https://marketplace.visualstudio.com/items?itemName=jbangdev.jbang-vscode)
- [IntelliJ](https://plugins.jetbrains.com/plugin/18257-jbang)

### エクスポート

フルスペックの開発にもすぐ移行できる

```console
jbang export maven app.java
jbang export gradle app.java
```

とはいえ、だんだんやってることが膨らんでくると、JBangにIDE、では限界が来るかもしれませんよね。

PoCのフェーズが終わって、本格的に開発を始めたくなったら、MavenやGradleのプロジェクトにエクスポートしてください。
これで、そのまま開発の続きを進められます。

## 実践編

1. REST
2. gRPC
3. AI / LLM

ここまでで基本的な部分は紹介しました。

ここからは、実際私がJBang使ってやってきたことをベースに、こんなことまでできますよ、というのを紹介していきたいと思います。

## CLI開発

```console
jbang init --template=cli cli.java
```

CLI開発についても触れておきましょう。

ここまで、スクリプトでこれだけのことができるとなったら、ちょっとした自動化ツールとか作ってみたくなりますよね。
シェルスクリプトでそういうコマンドラインツールを作るのは定番です。
JBangでちょっとしたCLIツールを作るというのは、実はかなり最適な使い方です。

### picocli

<https://picocli.info/>

JavaにはpicocliというCLI開発のためのフレームワークがあります。
JBangはこれととても相性がよくて、テンプレートまで用意されてます。
このコマンドひとつで、簡単にCLIスクリプトの雛形が生成できます。

### JBangベースのCLI

- [Camel](https://camel.apache.org/manual/camel-jbang.html) — Integration framework
- [Hawtio](https://hawt.io/docs/get-started.html) — Management console
- [Citrus](https://citrusframework.org/samples/jbang/) — BDD integration testing framework

ここからエコシステムの話につながってくるんですが、
実際にJBangを使ってJavaのフレームワークやアプリのCLIを作って行こうっていうトレンドがあるんです。

私の関わってるApache Camel、Hawtio、それにインテグレーションテスティングフレームワークのCitrusとか。
この辺のプロジェクトは、いま説明したJBangとpicocliを使って独自のCLIを提供しています。

### カタログ

`jbang-catalog.json`

<https://github.com/user/repo> にある場合

```console
jbang my-cli@user/repo
```

<https://github.com/user/jbang-catalog> にある場合

```console
jbang my-cli@user
```

なんでJBangを使ってCLIを提供するかというと、カタログという機能があるからなんです。

はじめにGist上のスクリプトも直接実行できるという話をしましたが、それと似たような仕組みで、`jbang-catalog.json`というファイルを定義したリポジトリをGitHubで公開しておくと、そのユーザー名とリポジトリ名を参照して直接そのスクリプトを呼び出せる機能です。

さらに、リポジトリー名を`jbang-catalog`にしておくと、`コマンド名@ユーザー名`だけでスクリプトを実行できます。

### 例

<https://github.com/apache/camel/blob/main/jbang-catalog.json>

実際にApache CamelのJBangカタログを見てみましょう。

### アプリとしてインストール

さらに、アプリという機能もあって、このカタログのスクリプトをコマンドとしてローカルにインストールできるんです。
これが、JBangがコード配布のエコシステムとして優れているという理由です。

```console
jbang app install my-cli@user/repo

my-cli
```

### 公式カタログ

最後に、主要なベンダーでもJBangカタログを公開する流れがちょっとづつ増えています。

```console
jbang catalog list
```

- [Alibaba](https://github.com/alibaba/jbang-catalog)
- [Apple](https://github.com/jbanghub/apple)
- [Microsoft](https://github.com/microsoft/jbang-catalog)
- [Oracle (GraalPy)](https://github.com/oracle/graalpython)
- ...

ぜひ日本からも、JBangカタログでJavaのコードを配布する動きが広まってくれるといいなと思っています。

## まとめ

今日の話はこれで終わりです。

冒頭ののび太のように、これからのJava開発、JBangメインでやっていけるんじゃね、と閃いてくれた方が少しでもいてくれたら嬉しいです。

とりあえず、JBangをインストールして触ってみるだけでも面白いので、ぜひ試してみてください。

本日の資料は私のGitHubのリポジトリにソースコードも含めて公開しています。

質問があればどうぞ。
