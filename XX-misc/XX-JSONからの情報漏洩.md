# 4.xx.x JSONからの情報漏洩

## 概要

SPAなどAjaxによる画面更新を実装するために JSON を返すアクションを作ることがあります。その際、オブジェクトを不用意にシリアライズしてしまうと情報漏洩につながる場合があります。

このテストでは、JSONにシリアライズする属性が最小限となっており、不必要な情報が公開されていないことを確認します。

## 静的テスト

コントローラを正規表現 render.+?json で検索し、必要最小現の属性のみシリアライズされていることを確認します。

### 脆弱なコードの例

非公開情報や秘密情報が含まれているオブジェクトを render json している場合、脆弱です。

```ruby
def get_user_info
  render json: @user
```

安全なコードの例
画面描画に必要な属性だけシリアライズしている場合、安全です。

```ruby
def get_user_info
  render json: @user.slice(:id, :name)
```

## 動的テスト

JSONを返すURLにアクセスし、本文に非公開情報が含まれていないことを確認します。

### 脆弱性がある応答の例

JSONに画面描画に不要な属性や非公開情報が含まれている場合、脆弱です。

```http
HTTP/1.1 200 OK
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Download-Options: noopen
X-Permitted-Cross-Domain-Policies: none
Referrer-Policy: strict-origin-when-cross-origin
Content-Type: application/json; charset=utf-8
​
{"id":1,"email":"admin@example.com","password":"c93ccd78b2076528346216b3b2f701e6","admin":true,"first_name":"Admin","last_name":"","created_at":"2021-03-30T13:11:59.226Z","updated_at":"2021-03-30T13:11:59.354Z","auth_token":"YuiYZyqVrlsLk7+6YcrC1Q=="}
```

### 脆弱性がない応答の例

JSONに画面描画に必要な情報しかない場合、安全です。

```http
HTTP/1.1 200 OK
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Download-Options: noopen
X-Permitted-Cross-Domain-Policies: none
Referrer-Policy: strict-origin-when-cross-origin
Content-Type: application/json; charset=utf-8

{"id":1,"first_name":"Admin"}
```