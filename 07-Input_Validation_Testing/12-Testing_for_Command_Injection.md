# 4.7.12 Testing for Command Injection

## 概要

コマンドインジェクションのテストです。

## テスト方法

OSコマンドを実行する `system` 等を使用しているソースコードをレビューします。ここれらのメソッドに動的パラメータを使用している場合、脆弱な可能性が高いため、動的テストでパラメータを検証します。

検索キーワードの例：`` `.+`|%x|system|open|exec``

## 脆弱なコードと攻撃パラメータの例

次のコードでは、任意のOSコマンドを実行できてしまいます。

```ruby
def test_command
    result = `ping #{params[:ip]}`
    render status: 200, plain: result
end
```

次のリクエストを送信すると `cat config/routes.rb` の実行結果を取得できます。

```http
GET /test_command?ip=127.0.0.1|cat%20config/routes.rb
```

### 安全なコードの例

入力値が十分に検証されている場合、安全です。

```ruby
require 'resolv'

def test_command
    return head 400 if Resolv::IPv4::Regex !~ ip_address(params[:ip])
    result = `ping #{params[:ip]}`
    render status: 200, plain: result
end
```

## 追加情報

RubyのOSコマンド実行方法
- https://docs.ruby-lang.org/ja/latest/doc/spec=2fliteral.html#command
- https://docs.ruby-lang.org/ja/latest/method/Kernel/m/exec.html
- https://docs.ruby-lang.org/ja/latest/method/Kernel/m/system.html
- https://docs.ruby-lang.org/ja/latest/method/Kernel/m/open.html
- https://docs.ruby-lang.org/ja/latest/class/Open3.html