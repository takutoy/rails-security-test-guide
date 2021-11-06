# Regular Expressions DoS

## 概要

Regular expression Denial of Service (ReDoS) とは、正規表現処理に大量の計算リソースを消費させる攻撃手法です。

正規表現が次の3つの特性を含む場合、正規表現の実装によっては、特定の文字列のマッチ処理で Catastrophic backtracking が発生して CPUリソースが大量に消費されます。 

1. グループに繰り返しが使われている：`(xxx)+` や `(xxx)*` のこと
2. グループの中で繰り返しまたは複数パターンでマッチする：`a+` または `aa|a` のようなパターン
3. 最後にグループの中でマッチしない文字列がある

具体的には次のような正規表現です。

- `(a+)+$`
- `([a-zA-Z]+)*123`
- `(a|aa)+c`
- `(a|a?)+!`

このような正規表現に `aaaaaaaaaaaaaaaaaaaa!` のような文字列をマッチさせようとすると、aの数を1つ増やすごとにマッチ処理の時間が約2倍に伸びていきます。つまりサーバーのリソースが大量に消費されます。

## テスト方法

正規表現を使っているコードをレビューし、次の2点を確認します。

1. Catastrophic backtracking が発生しうる正規表現でユーザ入力値をマッチ処理していないか
2. ユーザ入力値を使って正規表現を作成していないか

該当するコードがあった場合、動的テストでパラメータを検証し、応答時間を測定します。

※攻撃文字列は応答時間が数秒程度になるように、また、同時に複数のリクエストしないように注意しましょう。

### Catastrophic backtracking が発生しうる正規表現

入力値検証に正規表現を使用している場合は要注意です。正規表現が3つの特性を持っていないか確認します。

```ruby
# parameter validation
re = Regexp.new("^(a+)+$")
return head 403 unless re =~ params[:input]
```

このコードには次のようなリクエストを送信すると攻撃が成立します。

```http
/login?input=aaaaaaaaaa!
```

### ユーザ入力値で正規表現を作成している

正規表現インジェクション（って言うのかな？）が可能な場合、ReDoS攻撃に脆弱かもしれません。

```ruby
# check password strength
username = params[:username]
password = params[:password]
re = Regexp.new(username)
return render plain: "ユーザ名にパスワードを含めないでください" if re=~password
```

このコードには次のようなリクエストを送信すると攻撃が成立します。

```
/login?username=(a%2b)%2b$&password=aaaaaaaaaa!
```

## 参考情報

- [OWASP Regular expression Denial of Service - ReDoS](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS)
