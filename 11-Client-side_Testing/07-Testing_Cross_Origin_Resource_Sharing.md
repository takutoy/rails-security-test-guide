# 4.11.7 Cross Site Resource Sharing (CORS)

## 概要

CORSの設定ミスは Same Origin Policy によるセキュリティを緩めてしまい、意図しない情報漏洩につながる可能性があります。

このテストでは Access-Control-Allow-Origin ヘッダが適切に設定されていることを確認し、悪意あるサイトから不正にリソースを参照されないかを検証します。

※CORSの詳細はhttps://developer.mozilla.org/ja/docs/Web/HTTP/CORSを参照してください。

## 静的テスト

すべてのソースコードを Access-Control-Allow-Origin で検索し、Access-Control-Allow-Origin ヘッダに信頼できない Origin を設定していないことを確認します。

### 脆弱なコードの例

ワイルドカード

```ruby
headers['Access-Control-Allow-Origin'] = '*'
```

### Originリクエストヘッダを検証せずにコピー

```ruby
headers['Access-Control-Allow-Origin'] = request.headers["Origin"]
```

### null

```ruby
headers['Access-Control-Allow-Origin'] = 'null'
```

※非公開情報を含まない、オープンなAPIの場合のみ * を設定しても安全です。

## 安全なコードの例

### 信頼できる Origin

```ruby
headers['Access-Control-Allow-Origin'] = `https://our-website.example.com'
```

### 許可リストに基づいて検証された request.headers["Origin"]をコピー

```ruby
if %w(https://example.com https://example.jp).include?(request.headers["Origin"])
  headers['Access-Control-Allow-Origin'] = request.headers["Origin"]
end
```

## 動的テスト

リクエストのOriginヘッダをいくつかのパターンで送信し、レスポンスのAccess-Control-Allow-Origin ヘッダを観察します。

### テストするリクエストのパターン

A. Webサイトと同一のオリジン

```http
GET /api/me/accounts HTTP/1.1
Host: example.com
Origin: https://example.com
```

B. Webサイトとは無関係のオリジン

```http
GET /api/me/accounts HTTP/1.1
Host: example.com
Origin: https://evil.com
```

C. Webサイトと前方または後方または部分一致しているドメイン

```http
GET /api/me/accounts HTTP/1.1
Host: example.com
Origin: https://evilexample.com
```

D. nullオリジン

```http
GET /api/me/accounts HTTP/1.1
Host: example.com
Origin: null
```

### 脆弱なレスポンスの例

Access-Control-Allow-Origin ヘッダがワイルドカードの場合、脆弱です。

```http
HTTP/1.1 200 OK
Content-Type: application/javascript
Access-Control-Allow-Origin: *
```

Access-Control-Allow-Origin ヘッダがnullの場合、脆弱です。

```http
HTTP/1.1 200 OK
Content-Type: application/javascript
Access-Control-Allow-Origin: null
```

Access-Control-Allow-Origin ヘッダがリクエストの Origin ヘッダと同一の場合、脆弱です。

```http
HTTP/1.1 200 OK
Content-Type: application/javascript
Access-Control-Allow-Origin: https://evil.com
```

### 安全なレスポンスの例

Access-Control-Allow-Origin ヘッダがない場合、安全です。

```http
HTTP/1.1 200 OK
Content-Type: application/javascript
```

Access-Control-Allow-Origin ヘッダが、A, B, C, Dどれでも同じ場合、安全です。

```http
HTTP/1.1 200 OK
Content-Type: application/javascript
Access-Control-Allow-Origin: https://example.com
```