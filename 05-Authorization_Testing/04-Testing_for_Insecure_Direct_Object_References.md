# 4.05.04 Insecure Direct Object Reference (IDOR)

## 概要

URL等に含まれるリソースIDが改ざんされて他のユーザのリソースが不正に操作されてしまうことがあります。

このテストでは、リソースへのアクセスが適切に制御されていて、悪意あるユーザによる不正な参照や変更から保護されているか検証します。

[4.05.04 Bypassing Authorization Schema](02-Testing_for_Bypassing_Authorization_Schema.md) も併せて参照してください。

## 静的テスト

コントローラ、アクション、クエリに認可が含まれていることを確認します。

### 脆弱なコードの例

コントローラやアクションにアクセス制御がなく、クエリに認可が含まていない場合、脆弱であると判断します。

```ruby
class UsersController < ApplicationController
  def update
    @article = Articles.find(params[:article_id])
    @article.update!(article_params)
    redirect_to @article
  end
```

### 安全なコードの例

クエリに認可が含まれている場合、安全です。

```ruby
class ArticlesController < ApplicationController
  def update
    @article = @current_user.articles.find(params[:article_id])
    @article.update!(article_params)
    redirect_to @article
  end
```

横の権限昇格がない場合、コントローラやアクションでアクセス制御していれば安全です。

```ruby
class AdminsController < ApplicationController
  before_action :is_admin?
  def update
    @article = Articles.find(params[:article_id])
    @article.update!(article_params)
    redirect_to @article
  end
```

認可ライブラリに cancancan を使っている場合、下記をレビューします。

- アクションに `authorize!` があること
- Ability に定義された認可ロジックの妥当性

認可ライブラリに pundict を使っている場合、下記をレビューします。

- アクションに `authorize` があること
- Policy に定義された認可ロジックの妥当性

<!--
https://github.com/CanCanCommunity/cancancan
https://github.com/varvet/pundit
-->

## 動的テスト

URLやパラメータに含まれるリソースIDを変更し、ログインしているユーザが本来参照できないページが表示される場合、脆弱であると判断します。

元のURL

```
http://127.0.0.1:3000/users/8/messages
```

変更後のURL

```
http://127.0.0.1:3000/users/1/messages
```

GETメソッドでない場合、リクエスト本文に含まれるリソースIDを変更します。