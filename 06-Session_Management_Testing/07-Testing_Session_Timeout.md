# 4.06.07 Session Timeout

## 概要

TBD

## 静的テスト

セッションストレージの設定値を確認します。

- `ActionDispatch::Session::CookieStore`
- `ActionDispatch::Session::CacheStore`
- `ActionDispatch::Session::ActiveRecordStore`

### CookieStore

`config.session_store` に `expire_after` が設定されていることを確認します。

設定されていない

```ruby
config.session_store :cookie_store
```

設定されている

```ruby
config.session_store :cookie_store, expire_after: 1.days
```

### CacheStore

### ActiveRecordStore

### Devise

## 動的テスト


## references

- [Railsガイド - action controller - セッション](https://railsguides.jp/action_controller_overview.html#%E3%82%BB%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3)

- https://api.rubyonrails.org/v6.1.3.2/classes/ActionDispatch/Session/CookieStore.html