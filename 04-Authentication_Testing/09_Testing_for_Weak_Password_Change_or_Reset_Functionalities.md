# Testing for Weak Password Change or Reset Functionalities

## 概要

## 静的テスト

## 動的テスト

### セルフサービスパスワードリセット

登録したメールアドレス宛にパスワードリセット用のURLが乗ってるような場合

確認すること

- パスワードリセットトークンの有効期限が過ぎたら使えないこと
- パスワードリセットが完了した後、トークンが無効化されること（再利用できないこと）
- パスワードリセットトークンを使って別のユーザのパスワードをリセットできないこと
- 他人のメールアドレスでリセットできないか
- リセット用URLが乗ったメールにHostヘッダの値を使用していないか

```
Host: test.local
X-Forwarded-Host: test.local
X-Forwarded-Server: test.local
X-Host: test.local
```