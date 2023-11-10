# 4.8.1 Testing for Improper Error Handling

## 概要

不適切なエラーハンドリングのテストです。

## 詳細なエラーメッセージを出力

例外オブジェクトの出力は、意図しない情報漏洩につながります。

### 脆弱なコードの例

```ruby
def index
    users = User.all.pluck(:id, :name)
    render json: users
rescue => e
    render json: { error: e }
end
```

### 情報漏洩に発展する例

例えば次のようなタイプミスをした場合に User テーブルがごっそり漏洩します。（とても極端な例ですが）

```ruby
def index
    users = User.all.earger_load(:roles) # typo for .erger_load
    render json: users.pluck(:id, :name)
rescue => e
    render json: { error: e }, status: internal_server_error
end
```

応答の例

```json
 {
    "error": "undefined method `earger_load' for #\u003cActiveRecord::Relation [#\u003cUser id: 1, name: \"タナカ\", email: \"tanaka@example.com\", created_at: \"2023-11-11 00:00:00.0000000000 +0000 【省略】 ,\u003e]\u003e\nDid you mean?  eager_load\n               eager_load!" 
}
```