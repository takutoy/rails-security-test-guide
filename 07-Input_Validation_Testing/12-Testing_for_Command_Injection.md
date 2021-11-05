# 4.7.12 Testing for Command Injection

## 概要

コマンドインジェクションのテストです。

## 静的テスト

OSコマンドを実行する `system` 等を使用しているソースコードをレビューします。これらのメソッドに動的パラメータを使用している場合、脆弱な可能性が高いです。

検索キーワードの例：`` `.+`|%x|system|open|exec``

### 脆弱なコードの例

次のコードでは、任意のOSコマンドを実行できてしまいます。

```ruby
def test_command
    result = `ping #{params[:ip]}`
    render status: 200, plain: result
end
```

攻撃リクエストの例
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

## 動的テスト

ソースコードレビューやBrakemanでコマンドインジェクションの疑いがあるコードが見つかった場合、該当するアクション・パラメータをテストします。

## 追加情報

RubyのOSコマンド実行方法
- https://docs.ruby-lang.org/ja/latest/doc/spec=2fliteral.html#command
- https://docs.ruby-lang.org/ja/latest/method/Kernel/m/exec.html
- https://docs.ruby-lang.org/ja/latest/method/Kernel/m/system.html
- https://docs.ruby-lang.org/ja/latest/method/Kernel/m/open.html
- https://docs.ruby-lang.org/ja/latest/class/Open3.html