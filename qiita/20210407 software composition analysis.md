# Ruby on Rails 製 Web アプリケーションのセキュリティテストガイド (ソフトウェアコンポジション解析：SCA)

## これなに？

Ruby on Rails 製 Webアプリケーションのセキュリティテストをするためのガイドです。本ガイドは脆弱なRailsアプリである [RailsGoat](https://github.com/OWASP/railsgoat) を題材にセキュリティテストの手法を解説します。

本記事では、アプリケーションが依存するライブラリのバージョン情報をもとに既知の脆弱性を発見する手法である __ソフトウェアコンポジション解析(SCA)__ を扱います。

関連記事

- [Ruby on Rails 製 Web アプリケーションのセキュリティテストガイド (静的テスト：SAST)](https://qiita.com/takutoy/items/e1867b51a6c8a15a93c8)

### 対象読者

Ruby on Rails でプロダクト開発している組織の**セキュリティチーム**を想定していますが、セキュリティに関心がある**開発者**や**テスター**にとっても役立つしれません。

前提知識・スキル

- 情報セキュリティに関する基本的な用語を知ってること
- 基本的なWebアプリケーションの脆弱性について知ってること
- Ruby on Rails のコードが読めること
- Linux のコマンドを叩けること

### 目次

1. ソフトウェアコンポジション解析(SCA) とは
2. Gem の脆弱性調査
    - bundler-audit を使って Gem の脆弱性を検出する
    - Ruby on Rails: Security グループを参照し、脆弱性のプロダクトへの影響を調査する
3. Javascript ライブラリの脆弱性調査
    - Retire.js を使って Javascript ライブラリの脆弱性を検出する
    - 各種情報源を参照し、脆弱性のプロダクトへの影響を調査する

## ソフトウェアコンポジション解析(SCA) とは

どんなに注意深くソースコードを書いたとしても、脆弱性ゼロのWebアプリケーションを作ることはできません。なぜならアプリケーションを動かすためのライブラリやミドルウェアにも脆弱性は存在しうるからです。

Rails アプリケーションも多くの Gem や Javascript ライブラリを使って開発します。ソースコードに問題がなくても Gem に脆弱性があれば Web アプリケーションは危険にさらされることになります。

そのため、アプリケーションが依存する全てライブラリに存在する脆弱性も調査する必要があります。これをソフトウェアコンポジション解析 (Software Composition Analysis, SCA) と言います。

ソフトウェアコンポジション解析でプロダクトに存在する脆弱性を検出するには、アプリケーションが依存するライブラリとバージョンを列挙したあと、次の4ステップを確認します。

1. ライブラリに脆弱性があるか
2. バージョンに脆弱性があるか
3. 脆弱性を踏むコードがあるか
4. 回避策が実装されているか

![](https://takutoy.s3-ap-northeast-1.amazonaws.com/qiita/202103-rails-security-test-guide/2021-04-06-10-26-20.png)

本章で紹介する bundler-audit と Retire.js は、ライブラリとバージョンを列挙し存在する脆弱性を検出(①～②)してくれますが、プロダクトがその脆弱性の影響を受けるか否かの判断(③～④)はしてくれません。

セキュリティテストの目的によっては、バージョンチェック（①と②）のみ実施して古いライブラリのバージョンアップを促すだけで十分な場合もあれば、脆弱性の影響調査（③と④）も実施してバージョンアップの緊急度も含めて情報提供する方が良い場合もあります。目的に合った範囲を選びましょう。

### 影響調査の必要性

ライブラリに脆弱性があるからといって、必ずしもプロダクトが危険な状態であるとは限りません。ライブラリに脆弱性が存在したとしても、当該機能を使っていなければ影響は受けないからです。

脆弱性の影響を受けないにもかかわらず、今すぐライブラリをバージョンアップしろ！ASAP！と言うのはあまり望ましくはありません。

このような事態を避けるために、脆弱性のプロダクトへの影響を調査し、バージョンアップの緊急度や優先度を判断する情報を可能な限り提供できるようにします。

### 影響調査の手法

影響調査は静的・動的、両方の手法で実施可能です。

- 静的手法：当該機能が使用されているかをソースコードから確認する
- 動的手法：Webアプリに疑似攻撃を仕掛けて確認する

ライブラリがアプリケーションに直接依存している場合は静的手法が確認しやすいですが、間接的に依存している場合は動的手法のほうが容易になる場合もあります。

本章では静的手法で影響調査します。動的手法は動的アプリケーションセキュリティテスト(DAST)の章で扱います。

## Gem の脆弱性調査

Rails アプリケーションは標準でも多数の Gem に依存しています。本節では Gem の SCA ツールである [bundler-audit](https://github.com/rubysec/bundler-audit) を使って脆弱性一覧表を作成し、さらに脆弱性のプロダクトへの影響を調査します。

※bundler-audit のセットアップ手順等は公式サイトを参照してください

### bundler-audit を使って Gem の脆弱性を検出する

bundler-audit を使うには、Railsアプリケーションのソースコードがあるディレクトリに移動して `bundler-audit` コマンドを実行します。

が、実行する前に `bundler-audit update` で脆弱性データベースを更新しておきましょう。

実行例

```shell
user@sectest:~/railsgoat$ bundler-audit update
Updating ruby-advisory-db ...
From https://github.com/rubysec/ruby-advisory-db
 * branch            master     -> FETCH_HEAD
Already up to date.
Updated ruby-advisory-db
ruby-advisory-db:
  advisories:   485 advisories
  last updated: 2021-03-10 12:29:28 -0800
```

```shell
user@sectest:~/railsgoat$ bundler-audit
Name: actionpack
Version: 6.0.0
CVE: CVE-2020-8164
Criticality: Unknown
URL: https://groups.google.com/forum/#!topic/rubyonrails-security/f6ioe4sdpbY
Title: Possible Strong Parameters Bypass in ActionPack
Solution: upgrade to ~> 5.2.4.3, >= 6.0.3.1

Name: actionpack
Version: 6.0.0
CVE: CVE-2020-8166

...

Vulnerabilities found!
```

`bundler-audit` はこのように結果をプレーンテキスト形式で出力しますが、これでは記録しづらいので、スプレッドシートに貼り付けられる形式に変換しましょう。

`bundler-audit` は結果を CSV で出力することはできませんが、json で出力することはできます。

```
user@sectest:~/railsgoat$ bundler-audit --format json
{
  "version": "0.8.0",
  "created_at": "2021-03-14 15:29:19 +0900",
  "results": [
    {
      "type": "unpatched_gem",
      "gem": {
        "name": "actionpack",
        "version": "6.0.0"
      },
      "advisory": {
...
```

これを `jq` を使って TSV に変換すればスプレッドシートに貼り付けられるようになります。

サンプルスクリプト

```bash
bundle-audit check --format json \
| jq -r '["gem", "version", "advisory", "title", "url"], 
  (.results[] | 
    [
      .gem.name,
      .gem.version,
      .advisory.id,
      .advisory.title,
      .advisory.url
    ] 
  ) 
  | @tsv' \
| clip.exe
```

補足：

* `clip.exe` は出力をクリップボードにコピーする WSL のコマンドで、`pbcopy` や `xclip` みたいなモノです
* TSV に出力する項目は必要に応じて変えてください

bundler-audit の出力を貼り付けたスプレッドシートの例

![](https://takutoy.s3-ap-northeast-1.amazonaws.com/qiita/202103-rails-security-test-guide/2021-03-26-20-08-44.png)

これで結果を確認しやすくなり、影響調査の記録もしやすくなりました。

### 影響調査の流れ

Gem の脆弱性情報は [Ruby on Rails: Security グループ](https://groups.google.com/g/rubyonrails-security) に良く整理されているのでこれを参照します。`bundler-audit` の出力も、このグループへのリンクがほとんどです。

ここでは [CVE-2020-8164: Possible Strong Parameters Bypass in ActionPack](https://groups.google.com/g/rubyonrails-security/c/f6ioe4sdpbY) を例に、Railsgoatがこの脆弱性の影響を受けるか確認してみましょう。

ページを開くと次のような情報が出てきます。影響調査で特に重要な項目は Impact (影響) と Workaroud (回避策) です。

![](https://takutoy.s3-ap-northeast-1.amazonaws.com/qiita/202103-rails-security-test-guide/2021-04-06-18-20-52.png)

まずは Impact を読みましょう。

簡単に言うと `params.each` `params.each_value` `params.each_pair` はヤバい。言い換えると `params.each` が Railsgoat に含まれていなければ安全と判断できそうです。

Visual Studio Code でソースコード全体を `params.each` で検索してみましょう。

![](https://takutoy.s3-ap-northeast-1.amazonaws.com/qiita/202103-rails-security-test-guide/2021-04-06-18-20-15.png)

マッチしませんでした。影響なし！

今回は影響ありませんでしたが、もしマッチした場合は、該当箇所に Workaround があるかを確認します。Workaround には

```
Do not use the return values of `each`, `each_value`, or `each_pair` in your
application.
```

とあるので `params.each` `params.each_value` `params.each_pair` の戻り値を使ってなければセーフ（影響なし）、使ってたらアウト（影響あり）、という判断ができます。

このような流れで、全ての脆弱性の Impact と Workaround を確認します。

影響調査の記録例：

![](https://takutoy.s3-ap-northeast-1.amazonaws.com/qiita/202103-rails-security-test-guide/2021-04-06-18-19-40.png)

## Javascriptライブラリの脆弱性調査

Rails アプリケーションは Gem だけでなく、多くの場合 javascript ライブラリにも依存します。本節では Javascript の SCA である [Retire.js](https://github.com/retirejs/retire.js/) を使って脆弱性を検出し、脆弱性一覧表を作成します。

※Retire.js のセットアップ手順等は公式サイトを参照してください

### Retire.js を使って Javascript ライブラリの脆弱性を検出する

Retire.js を使うには、Rails アプリケーションのソースコードがあるディレクトリに移動して `retire` コマンドを実行します。

コマンド実行例

```
user@sectest:~/railsgoat$ retire
retire.js v2.2.4
Downloading https://raw.githubusercontent.com/RetireJS/retire.js/master/repository/jsrepository.json ...
Downloading https://raw.githubusercontent.com/RetireJS/retire.js/master/repository/npmrepository.json ...
/home/user/railsgoat/app/assets/javascripts/jquery.min.js
 ↳ jquery 1.8.3
jquery 1.8.3 has known vulnerabilities: severity: medium; CVE: CVE-2012-6708, bug: 11290, summary: Selector interpreted as HTML; http://bugs.jquery.com/ticket/11290 https://nvd.nist.gov/vuln/detail/CVE-2012-6708 http://research.insecurelabs.org/jquery/test/ severity: medium; issue: 2432, summary: 3rd party CORS request may execute, CVE: CVE-2015-9251; https://github.com/jquery/jquery/issues/2432 http://blog.jquery.com/2016/01/08/jquery-2-2-and-1-12-released/ https://nvd.nist.gov/vuln/detail/CVE-2015-9251 http://research.insecurelabs.org/jquery/test/
...
```

すごく、見づらいです。。。

`retire` は `bundler-audit` と同様に、結果を json で出力させることができます。`jq` で TSV に整形してスプレッドシートに貼り付けましょう。

サンプルスクリプト

```bash
retire --outputformat json 2>&1 \
| jq -r '["component", "version", "advisory", "title", "url", "file"], 
  (.data[] | . as $data | .results[] | . as $result | .vulnerabilities[] | 
    [
      $result.component,
      $result.version,
      (.identifiers.CVE | join(", ")),
      .identifiers.summary,
      (.info | join(", ")),
      $data.file
    ]
  )
  | @tsv' \
| clip.exe
```

* `clip.exe` はクリップボードにコピーするWSLのコマンドで、`pbcopy` や `xclip` みたいなモノです
* TSV に出力する項目は必要に応じて変えてください

`retire` の出力を貼り付けたスプレッドシートの例

![](https://takutoy.s3-ap-northeast-1.amazonaws.com/qiita/202103-rails-security-test-guide/2021-03-26-20-21-25.png)

### 脆弱性の影響調査

Javascript ライブラリの脆弱性も、Gem の脆弱性と同様に必ずしもプロダクトに影響を与えるわけではないので影響調査をします。

調査には Retire.js が出力したリンク先を参照し、影響を受ける使い方や回避策を探せばよいです。

ただし Javascript ライブラリの脆弱性の情報は、Gem と比較すると整理されていないことが多いため、情報が無かったり見つけにくいこともあります。調査に時間がかかりそうなら、後回しや中断することも検討しましょう。

筆者はまず比較的情報が整理されている [NVD (https://nvd.nist.gov/)](https://nvd.nist.gov/) を参照し、次にその他の情報源を当たることが多いです。

### Retire.js の注意点

Retire.js はライブラリやバージョン情報をファイル名やコンテンツ等から抽出しているため、何らかの理由でファイルが変更されてたりすると脆弱性が検出できない場合もあります。

実はこの問題は Railsgoat の例でも発生しています。Railsgoat で使われている moment.js 1.7.2 には [ReDoSの脆弱性](https://github.com/moment/moment/issues/2936) が存在しますが、Retire.js は検出していません。

Retire.js は moment.js の脆弱性を検出しなかった図：

![](https://takutoy.s3-ap-northeast-1.amazonaws.com/qiita/202103-rails-security-test-guide/2021-04-06-16-09-16.png)

![](https://takutoy.s3-ap-northeast-1.amazonaws.com/qiita/202103-rails-security-test-guide/2021-03-26-20-21-25.png)

このような見逃しを許容できない場合、Javascriptファイルを目視で確認したり、開発ドキュメントを参照して使用されているライブラリとバージョンを二重でチェックしておくとよいでしょう。

## ソフトウェアコンポジション解析まとめと補足

本記事ではソフトウェアコンポジション解析(SCA)ツールである bunlder-audit と Retire.js を使って、Railsアプリケーションが依存しているライブラリの既知の脆弱性を検出しました。

また、検出された脆弱性がプロダクトへの影響を調査するために、[Ruby on Rails: Security グループ](https://groups.google.com/g/rubyonrails-security) や  [NVD](https://nvd.nist.gov/) を使いました。

Railsアプリケーションは多くのGemやJavascriptライブラリに依存しており、ライブラリは日々新しい脆弱性が報告されているため、定期的にソフトウェアコンポジション解析を実施するのが望ましいです。

ただしこれはセキュリティテストを頻繁に行うというよりも、開発チームがライブラリの脆弱性に早く気付く仕組み作るほうが良いでしょう。例えば `bundler-audit` や `retire` を CI に組み込んでみるとか、[Dependabot](https://docs.github.com/ja/code-security/supply-chain-security/configuring-dependabot-security-updates) の利用を促してみるのも手です。

## 最後に

本記事は現在執筆中の [Ruby on Rails 製 Web アプリケーションのセキュリティテストガイド](https://github.com/takutoy/rails-security-test-guide) を Qiita 向けに改編したものです。

今後、別記事で下記のトピックについても扱う予定です。

- セキュリティテストの計画と準備
- 動的アプリケーションセキュリティテスト(DAST)
- セキュリティテスト結果報告書の作成

本ガイドは常に改善を必要としています。お気づきの点があったらフィードバックを頂けるととても喜びます。