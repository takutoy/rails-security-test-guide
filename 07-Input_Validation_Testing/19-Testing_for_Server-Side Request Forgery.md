# 4.7.19 Testing for Server-Side Request Forgery

## 概要

SSRFのテストです。

## テスト方法

HTTP リクエストを実行しているコードをレビューします。URLに動的パラメータが含まれる場合、脆弱な可能性が高いです。

検索キーワードの例：`Net::HTTP|RestClient|HTTPX?\.|Faraday\.`

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
