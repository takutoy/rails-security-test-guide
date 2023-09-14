# 4.06.07 Session Timeout

## 概要

セッションの無効化が適切に実行されることを確認します。

- ログアウト
- タイムアウト
- 権限の変更・無効化

## 静的テスト

セッションストレージの設定値を確認します。

- `ActionDispatch::Session::CookieStore`
- `ActionDispatch::Session::CacheStore`
- `ActionDispatch::Session::ActiveRecordStore`

CookieStoreを使用している場合、次の事項に注意してください。

### CookieStore

CookieStore ではログアウト（サーバーサイドでのセッション無効化）ができません。サーバーサイドでのセッション無効化が必須の場合は CacheStore または ActiveRecordStore を使用してください。CookieStore を使う場合は、セッションの有効期間を設定してリスクを低減できます。

有効期限は `config.session_store` に `expire_after` が設定されているかで確認できます。

設定されていない場合：

```ruby
config.session_store :cookie_store
```

設定されている場合：

```ruby
config.session_store :cookie_store, expire_after: 1.days
```

## 動的テスト

- ログアウト後にログアウト前のセッションIDを使用してみる

## その他

管理者により権限が変更されたときに、既存のセッションの権限も変更されるか。

例：
- 管理者によりユーザが無効化されたとき、ユーザのセッションは無効化されるか
- 管理者によりテナントが無効化されたとき、テナントのユーザのログインやセッションは無効化されるか
- 管理者によりテナントが無効化されたとき、テナントのユーザはパスワードリセットができるか

## references

- [Railsガイド - action controller - セッション](https://railsguides.jp/action_controller_overview.html#%E3%82%BB%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3)

- https://api.rubyonrails.org/v6.1.3.2/classes/ActionDispatch/Session/CookieStore.html