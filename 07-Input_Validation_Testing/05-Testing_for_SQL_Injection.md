# 4.7.5 Testing for SQL Injection

## 概要

SQLインジェクションのテストです。

## 静的テスト

SQLを発行しているソースコードをレビューします。動的パラメータを文字列操作してSQLクエリを作成している場合、脆弱な可能性が高いです。

### 脆弱なコードの例

```ruby
User.where("id = '#{params[:id]}'").first
```

### 安全なコードの例

`?` を使い、パラメータを配列で渡している場合、安全です。

```ruby
User.where("id = ?", params[:id]).first
```

プレースホルダを使い、パラメータをハッシュで渡している場合、安全です。

```ruby
User.where("id = :user_id", { user_id: params[:id] }).first
```

ハッシュを使っている場合、安全です。

```ruby
User.where(id: params[:orders]).first
```

find や find_by_xxx を使っている場合、安全です。

```ruby
User.find(params[:id])
```

パラメータがサニタイズされている場合、適切な[サニタイズメソッド](https://api.rubyonrails.org/classes/ActiveRecord/Sanitization/ClassMethods.html)が使用されていることを確認してください。判断が難しい場合は動的テストの実施を推奨します。

### Brakeman

[Brakeman](https://brakemanscanner.org/) は Ruby on Rails 用の静的アプリケーションセキュリティテストツールで、脆弱性の疑いがあるソースコードを検出してくれます。

Brakeman はSQLインジェクションだけでなく、クロスサイトスクリプティングやオープンリダイレクトなど、様々な脆弱性にも対応しています。

Brakeman のインストールと使い方は [GitHubリポジトリのREADME](https://github.com/presidentbeef/brakeman) を参照してください。

Brakeman を使ってSQLインジェクションが検出した例

```shell
~railsgoat/$ brakeman
Loading scanner...

...[省略]...

Confidence: High
Category: SQL Injection
Check: SQL
Message: Possible SQL injection
Code: User.where("id = '#{params[:user][:id]}'")
File: app/controllers/users_controller.rb
Line: 52

...[省略]...
```

なお、Brakeman は見逃しや誤検知がある点は注意してください。

## 動的テスト

ソースコードレビューやBrakemanでSQLインジェクションの疑いがあるコードが見つかった場合、該当するアクション・パラメータをテストします。

手動でSQLインジェクションを試みるのはとても手間がかかるので、ツールを使うとよいでしょう。

### sqlmap

[sqlmap](https://sqlmap.org/) はSQLインジェクションの自動テストツールです。

sqlmap のインストールと使い方は [GitHubリポジトリのREADME](https://github.com/presidentbeef/brakeman#installation) を参照してください。

sqlmap の実行例

```shell
$ ./sqlmap.py -u 'http://127.0.0.1:3000/foobars?a=1&b=2&c=3'
```

sqlmap はテストするパラメータを自動的に判定してくれますが、パラメータの位置とDBMSを指定すると不要なテストを省略できます。

下記は DBMS が `postgresql` で、怪しいパラメータが `params[:a]` と `params[:c]` であると分かっている場合の実行例です。

```shell
$ ./sqlmap.py -u 'http://127.0.0.1:3000/foobars?a=*&b=2&c=*' --dbms=postgresql
```