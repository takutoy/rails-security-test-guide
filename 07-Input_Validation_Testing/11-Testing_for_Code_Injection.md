# 4.7.11 Testing for Code Injection

## 概要

コードインジェクションのテストです。

## テスト方法

動的にオブジェクトを生成したりメソッドを実行できる `send`, `constantize`, `eval` を使用しているソースコードをレビューします。これらのメソッドに動的パラメータを使用している場合、脆弱な可能性が高いため、動的テストでパラメータを検証します。

検索キーワードの例：`send|constantize|eval`

## 脆弱なコードと攻撃パラメータの例

### send

次のコードでは、Userクラスの任意のメソッドを実行できてしまいます。

```ruby
def test_send
    info = current_user.send(params[:method])
    render status: 200, json: info
end
```

次のリクエストを送信すると `current_user.reset_password` が実行できます。

```http
GET /test_send?method=reset_password
```

### constantize

次のコードでは、任意のクラスのfindメソッドを実行できてしまいます。

```ruby
def test_const
    user = params[:src].constantize.find(params[:id])
    render status: 200, json: user
end
```

次のリクエストを送信すると `SecretTable.find(1)` の結果を参照できます。

```http
GET /test_const?src=SecretTable&id=1
```

### eval

次のコードでは、任意のRubyコードを実行できてしまいます。

```ruby
def test_eval
    user = eval("User.find(#{params[:id]})")
    render status: 200, json: user
end
```

次のリクエストを送信すると `sleep(5)` を実行できます。
```http
GET /test/test_eval?id=1)%3Bsleep(5
```

## 安全なコードの例

動的パラメータを直接使用していない場合、安全です。

```ruby
mode = params[:method] == 1 ? 'first' : 'last'
@user = User.send(mode)
```

許可リスト方式でパラメータを検証している場合、安全です。

```ruby
return head 400 unless %w[User Customer].include?(params[:src])
```

## 追加情報

各種APIのドキュメント

- send: https://docs.ruby-lang.org/ja/latest/method/Object/i/send.html
- constantize: https://api.rubyonrails.org/classes/String.html#method-i-constantize
- eval: https://docs.ruby-lang.org/ja/latest/method/Kernel/m/eval.html