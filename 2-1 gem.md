## Gemの脆弱性を検出 (bundler-audit)

### bundler-audit

`bundler-audit` を使います。

コマンド実行前に、脆弱性データベースを更新しておきましょう。`bundler-audit update` 


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

### スプレッドシートに貼り付ける

脆弱性一覧がプレーンテキストだと管理しづらいので、スプレッドシートに貼り付けたいですね。

残念ながら今のところ結果をCSVで出力することはできませんが、jsonで出力することはできます。

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

これを `jq` を使って必要な項目を抽出してTSVに変換すれば、スプレッドシートに貼り付けられるようになります。

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

* 出力する項目は必要に応じて変えてください
* clip.exe はクリップボードにコピーするWSLのコマンドで、`pbcopy` や `xclip` みたいなモノです

`bundler-audit`の出力を貼り付けたスプレッドシートの例



### 脆弱性の精査

ライブラリに脆弱性がある、プロダクトがその脆弱性の影響を受けるか？を確かめる必要がある。これは2-3で説明します。