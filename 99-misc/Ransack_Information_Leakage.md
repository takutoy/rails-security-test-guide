# Ransack からの情報漏洩

## 概要

​Ransack は高度な検索機能を容易に実装できるGemです。

Ransack で検索可能な属性を制限していない場合、検索結果の差異から非公開の属性値を特定される危険性があります。

このテストでは Ransack で検索可能な属性が制限されており、情報漏洩から保護されているかを検証します。

## 静的テスト

次の3点を確認します。すべてが満たされている場合、脆弱です。

1. Ransack を使用している（Gemfileにransack が含まれている）
2. ransackメソッドにユーザ入力値を使用している
3. モデルにransackable が定義されていない

### 脆弱なコードの例

次のようなスキーマとUserモデルがあるとします。

```ruby
# db/schema.db
create_table "users", force: :cascade do |t|
  t.string "email"
  t.string "password"
  t.string "api_key"
  t.boolean "admin"
  t.string "first_name"
  t.string "last_name"
  t.datetime "created_at"
  t.datetime "updated_at"
end
```

`ransackable` による認可が定義されてなく、かつ `ransack` している場合、非公開の属性(例えばpasswordやapi_key)で検索できてしまうため、脆弱です。 

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # def ransackable_* がない
end
app/controllers/users_controller.rb
def search
  @q = User.ransack(params[:q])
```

### 安全なコードの例

ransack にユーザ入力値を使っていない場合、安全です。

```ruby
# app/controllers/users_controller.rb
def search
  @q = User.ransack(id_eq: 1)
```

`ransackable_attribute`, `ransackable_associations`, `ransortable_attributes` がモデルに定義されており、必要な属性・アソシエーションに絞られている場合も安全です。

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def self.ransackable_attributes(auth_object = nil)
    super & %w(email first_name last_name)
  end
​
  def self.ransortable_attributes(auth_object = nil)
    ransackable_attributes(auth_object)
  end
​
  def self.ransackable_associations(auth_object = nil)
    []
  end
end
```

モデルにアソシエーションや非公開の属性が含まれていない場合も安全です。

```ruby
# db/schema.db
create_table "users", force: :cascade do |t|
  t.string "email"
  t.string "first_name"
  t.string "last_name"
  t.datetime "created_at"
  t.datetime "updated_at"
end
```

## 動的テスト

ransackが使われているURLのクエリを操作し、非公開や秘密と思われる属性で検索します。応答の差異から値が特定できる場合、脆弱です。ransack クエリについては Ransack: Search Matchers を参照してください。

次のような、名前でユーザを検索できる機能を想定します。

http://127.0.0.1:3000/users/search?q[name_cont]=admin

`q[name_cont]=admin` は name 属性に'admin'が含まれているレコードを抽出するためのクエリで、`WHERE "users"."name" LIKE '%admin%'` のようなSQLに変換されます。

### 脆弱性がある場合の例

ransackはクエリに検索できない属性が指定された場合、そのクエリを無視します。その特性を利用して検索できる属性を特定することができます。

|URLとクエリ|件数|
|:--|:--|
|http://127.0.0.1:3000/users/search?q[name_eq]=not_exists_value|0|
|http://127.0.0.1:3000/users/search?q[email_eq]=not_exists_value|0|
|http://127.0.0.1:3000/users/search?q[address_eq]=not_exists_value|10|
|http://127.0.0.1:3000/users/search?q[password_eq]=not_exists_value|0|

このように検索結果の件数に差異が発生します。0件の場合その属性は検索可能、0件でない場合その属性は検索不可（または存在しない）であることが分かります。この例では name, email, password は検索可能、address は検索不可であることを示しています。

passwordは秘密情報であり検索できてはいけないので、上記の場合は脆弱です。なお、emailが公開情報であるかはアプリケーションの仕様によりますが、非公開情報の場合は脆弱と判断します。

### 脆弱性がない場合の例


|URLとクエリ|件数|
|:--|:--|
|http://127.0.0.1:3000/users/search?q[name_eq]=not_exists_value|0|
|http://127.0.0.1:3000/users/search?q[email_eq]=not_exists_value|10|
|http://127.0.0.1:3000/users/search?q[address_eq]=not_exists_value|10|
|http://127.0.0.1:3000/users/search?q[password_eq]=not_exists_value|10|

公開されている属性以外で検索ができない場合、安全です。