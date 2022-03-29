# 4.06.05 Cross Site Request Forgery (CSRF)

## 概要

Railsでは標準でCSRF攻撃の対策が実装されているため、正しく使っていればCSRFの脆弱性が作りこまれることはありません。

ただしCSRF対策機能を無効化していたり、RESTに沿わない実装をしている場合、CSRF攻撃に対して脆弱性になる場合があります。

このテストでは、RailsのCSRF対策機能が無効化されているような実装になっていないことを確認し、CSRF攻撃から保護されているかを検証します。

## 静的テスト

RailsにおいてCSRF攻撃に脆弱な実装には次のようなパターンがあります。

### CSRF保護を無効化している

`config/application.rb` に次の設定がある場合、CSRF保護が無効化されているため脆弱です。

```ruby
config.action_controller.allow_forgery_protection = false
```

`ActionController:Base` にCSRF保護を追加しない設定がある場合も脆弱です。

```ruby
config.action_controller.default_protect_from_forgery = false
```

<!--
### ActionController::APIでセッションCookieを使ってる

APIでセッションCookieを使う場合、CSRFに脆弱です。
-->


### CSRFトークン検証を省略または弱い設定にしている

更新系のアクションでトークン検証を省略している場合、脆弱です。

```ruby
class HogeController < ApplicationController
  protect_from_forgery :except => [:index, :show, :update]
```

トークン検証の省略には、次のようなパターンもあります。

```ruby
class HogeController < ApplicationController
  skip_forgery_protection
```

```ruby
class HogeController < ApplicationController
  skip_before_action :verify_authenticity_token
```

ログインしなくても実行可能なアクションにCSRF対策が必要な場合、null_sesion や reset_session は NG です。

```ruby
class OtoiawaseController < ApplicationController
  protect_from_forgery
  # protect_from_forgery with: :null_session と同等です
```

```ruby
class OtoiawaseController < ApplicationController
  protect_from_forgery with: :reset_session
```

### CSRFについて補足

CSRF攻撃はセッション Cookie に SameSite=Lax を明示的に設定することでも緩和可能です。

2022年3月現座、ほとんどのWebブラウザは SameSite の既定値を Lax としているため、CSRF 攻撃は困難な状況となっています。（SameSite の既定値が None の主要ブラウザは Safari と Firefox のみです）
参照 [SameSite cookies - HTTP | MDN：ブラウザの互換性](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Set-Cookie/SameSite#%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%81%AE%E4%BA%92%E6%8F%9B%E6%80%A7)

ただし次のようなアクションには SameSite=Lax が設定されていても攻撃は成立します。

- リソースを変更するGETメソッドなアクション
- ログインしなくても実行可能なアクション（匿名掲示板やお問い合わせフォームなど）

## 動的テスト

RailsではCSRFトークンをリクエスト本文の authenticity_token か、リクエストヘッダの X-CSRF-TOKEN で送信します。

更新系のリクエストから authenticity_tokenとX-CSRF-TOKENを消してもリクエストが受理された場合、そのアクションはCSRF攻撃に脆弱です。

元のリクエスト
![](images/2021-05-10-21-59-39.png)

変更後のリクエスト
![](images/2021-05-10-21-59-45.png)