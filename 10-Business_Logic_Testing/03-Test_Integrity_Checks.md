# 4.10.3 Test Integrity Checks

## 概要

Rails には Mass Assignment による不正なデータ改ざんを防ぐために、Strong Parameters という仕組みが導入されています。開発者は Strong Parameters を適切に使用することで、ユーザが変更可能な Active Model の属性を制限することができます。

しかし Strong Parameters の設定が不適切だと、依然としてデータ改ざんに対して脆弱な状態になってしまいます。

このテストでは Strong Parameters が適切に設定されていることを確認し、不正なデータ改ざんから保護されているかを検証します。

## 静的テスト

コントローラのソースコードで Strong Parameters を使用している箇所をレビューし、許可されている属性が最小限になっていることを確認します。

Strong Parameters を使用しているコードは `.permit` で検索すると見つけやすいでしょう。

![](images/2021-05-20-21-45-56.png)

### 脆弱なコードの例

全ての属性を許可：これは Mass Assignment を許可する危険な設定です。

```ruby
params.require(:user).permit!
```

不要なパラメータを許可

```ruby
params.require(:user).permit(:email, :admin, :first_name, :last_name)
```

### 安全なコードの例

必要最小限の属性のみ許可

```ruby
params.require(:user).permit(:email, :first_name, :last_name)
```

ユーザの権限によって許可する属性を切り替え

```ruby
params.require(:user).permit(:email, :first_name, :last_name, :admin) if user.admin
```
## 動的テスト

Create, Update リクエストのパラメータを書き換え、改ざんしたいパラメータを付与して送信します。

下図は Railsgoat の Account settings 画面 `http://127.0.0.1:3000/users/9/account_settings` です。

![](images/2021-05-20-22-05-48.png)

元のリクエスト

```http
POST http://127.0.0.1:3000/users/9.json HTTP/1.1
=== truncated ===

utf8=%E2%9C%93&_method=patch&authenticity_token=0z21aWys7V6qyJIHnfeJUA%2F9ivbhuYfB58YWX%2Fi5wKTTwT1YzrEBFxVETMxmynn%2BErkT0n%2FtPpf3bwK735k2lw%3D%3D&user%5Bid%5D=9&user%5Bemail%5D=b%40example.com&user%5Bfirst_name%5D=tanaka&user%5Blast_name%5D=ichiro&user%5Bpassword%5D=&user%5Bpassword_confirmation%5D=
```

改ざん後のリクエスト

`&user%5Badmin%5D=true` を追加します。

```http
POST http://127.0.0.1:3000/users/9.json HTTP/1.1
=== truncated ===

utf8=%E2%9C%93&_method=patch&authenticity_token=0z21aWys7V6qyJIHnfeJUA%2F9ivbhuYfB58YWX%2Fi5wKTTwT1YzrEBFxVETMxmynn%2BErkT0n%2FtPpf3bwK735k2lw%3D%3D&user%5Bid%5D=9&user%5Bemail%5D=b%40example.com&user%5Bfirst_name%5D=tanaka&user%5Blast_name%5D=ichiro&user%5Bpassword%5D=&user%5Bpassword_confirmation%5D=&user%5Badmin%5D=true
```