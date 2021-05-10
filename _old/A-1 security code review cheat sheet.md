# Rails セキュリティコードレビューの観点集

## クエリに認可が含まれていること

クエリに認可が含まれていない場合、URL等に含まれるリソースIDが改ざんされて他のユーザのリソースが不正に操作されてしまうことがあります。

例）自分のプロフィール画面のURL：http://example.com/customers/123 の `123` を `444` に変えてみたら、他人のプロフィールが見えた！

### レビュールール

コントローラを正規表現 `params\[:.*id\]` で検索し、リソースにアクセス制御が必要な場合、アクションまたはクエリにアクセス権が含まれているかを確認する。

### 悪い例

全てのリソースを更新できてしまう

```ruby
def update
  @article = Articles.find(params[:article_id])
  @article.update!(article_params)
  redirect_to @article
end
```

ただし下記の場合は問題ありません。

- コントローラ、アクションにアクセス制御が実装している（例えば管理者のみ使用できる、など）
- cancancan や pundit でアクセス制御を実装している

※アクセス制御ロジックの妥当性は別途確認しましょう

### 良い例

クエリにアクセス制御が含まれている

```ruby
def update
  @article = @current_user.articles.find(params[:article_id])
  @article.update!(article_params)
  redirect_to @article
end
```

## CORS が適切に構成されていること

CORSの設定ミスは Same Origin Policy によるセキュリティを緩めてしまい、意図しない情報漏洩につながる可能性があります。

CORSについての詳細は[MDN: オリジン間リソース共有 (CORS)](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS)を参照してください。

### レビュールール

すべてのコードを `Access-Control-Allow-Origin` で検索し、信頼できない Origin を設定していないことを確認します。

### 悪い例

ワイルドカード

```ruby
headers['Access-Control-Allow-Origin'] = '*'
```

※公開APIの場合のみ、ワイルドカードはOKです

リクエストヘッダをコピー

```ruby
headers['Access-Control-Allow-Origin'] = request.headers["Origin"]
```

null

```ruby
headers['Access-Control-Allow-Origin'] = 'null'
```

### 良い例

信頼できる Origin

```ruby
headers['Access-Control-Allow-Origin'] = `our-website.example.com'
```

許可リストに基づいて検証された `request.headers["Origin"]`

```ruby
if %w(example.com example.jp).include?(request.headers["Origin"])
  headers['Access-Control-Allow-Origin'] = request.headers["Origin"]
end
```

### 補足

凝った設定が必要な場合は[`rack-cors`](https://github.com/cyu/rack-cors)を使うのもよいと思います。

`rack-cors` を使っている場合も、Origin の妥当性は確認しましょう。

## JSON からの情報漏洩

SPAなどAjaxによる画面更新を実装するために、JSONを返すアクションを作ることがあります。その際、オブジェクトを不用意にシリアライズしてしまうと意図しない情報漏洩につながります。

### レビュールール

コントローラを `render json` で検索し、必要最小現の属性のみシリアライズされていることを確認します。

### 悪い例

オブジェクトをそのまま render

```ruby
render :json => @user
```

### 良い例

画面描写に必要な属性だけシリアライズ

```ruby
render :json => { :id => @user.id, :first_name => @user.first_name }
```

### 補足

jb や jbuilder を使っている場合、テンプレートに不要な属性が含まれていないことを確認してください。

## Strong Parameters が適切に設定されていること

Strong Parameters で更新可能なパラメータを不用意に permit してしまうと、開発者が意図しないデータを変更・改ざんされる危険性があります。

### レビュールール

すべてのコントローラを文字列 `.permit` で検索し、不要な属性が含まれていないことを確認します。

### 悪い例

全ての属性を permit

```ruby
params.require(:user).permit!
```

不要な属性を permit

```ruby
params
.require(:user)
.permit(:email, :admin, :first_name, :last_name)
```

### 良い例

必要最小限の属性だけ permit

```ruby
params
.require(:user)
.permit(:email, :first_name, :last_name)
```

## レースコンディション：ファイルパスが重複しないこと

一時ファイルなどを固定のファイルパスに書き込みしている場合、同時に複数のリクエストが実行されるとデータが混ざってしまいます。これによってデータの整合性がとれなくなったり、意図しない情報漏洩につながったしります。

### レビュールール

すべてのコードを正規表現 `File.open\(.+?"w` で検索し、ファイルパスが重複する可能性がないことを確認します。

※概要するロジックがアクションから呼び出されない場合は安全です

### 悪い例

固定値

```ruby
f = File.open("tmp/upload-file", "w")
```

ユーザがアップロードしたファイルの名前

```ruby
file = fileupload_param[:file]
output_path = Rails.root.join('public', file.original_filename)
```

### 良い例

```ruby

```

アクションから呼び出されない場合も安全です。

## レースコンディション：時刻をIDにしていないこと

ファイルだけでなく、IDやキーも同様に重複すると競合が発生します。見受けられるケースとしては、時刻をそのままIDとして

時刻をIDとして使ってしまうケースもあります。利用者が少数の時は発現する可能性は低いですが、利用者が増えてくるとリクエストが同時に処理される可能性が高くなり、複数のリクエスト間で同一のIDを操作してしまいます。これによってデータの整合性がとれなくなったり、意図しない情報漏洩につながったしります。

### レビュールール

すべてのコードを正規表現 `Time\.(now|current|zone\.now)` で検索し、IDやファイル名などを現在時刻だけで作成していないか確認する。

※概要するロジックがアクションから呼び出されない場合は安全です

### 悪い例

IDが時刻のみで構成されている

```ruby
f = File.open("tmp/#{Time.zone.now.to_i}", "w")
```

### 良い例

IDが時刻＋ユニークな値で構成されている

```ruby
f = File.open("tmp/#{Time.zone.now.to_i}_#{SecureRandom.uuid}", "w")
```

※一時ファイルを作るなら`tempfile`を使いましょう

## CSRF対策（GETメソッドでリソースを変更していないこと）

GETメソッドのアクションでリソースの状態を変更してはいけません

### レビュールール

GETメソッドでリソースを変更していないことを確認します。どのアクションがGETメソッドかは `rails routes` を見ると良いでしょう。

### 悪い例

GETメソッドのアクションでデータを作成、更新、削除をしている場合、CSRF攻撃に脆弱です。

```ruby
# GET /articles/:id/change_to_private
def change_to_private
    @article = @current_user.articles.find(params[:id])
    @article.private = true
    ...
```

### 良い例

GETメソッドのアクション内がデータの参照のみの場合、安全です。

```ruby
def show
    @article = @current_user.articles.find(params[:id])
end
```

## CSRF対策（トークン検証を省略していないこと）

CSRFトークン検証を省略してないこと。

XHR で `Can't verify CSRF token authenticity` エラーが出るからと言って安易にCSRFトークン検証を無効化してはいけません。

[](https://railsguides.jp/security.html#csrf%E3%81%B8%E3%81%AE%E5%AF%BE%E5%BF%9C%E7%AD%96)
https://guides.rubyonrails.org/security.html#csrf-countermeasures


### レビュールール

コントローラを次の文字列で検索し、不用意にトークン検証を無効化していないことを確認。

- `verify_authenticity_token`
- `skip_forgery_protection `
- `protect_from_forgery`

<!--
https://api.rubyonrails.org/classes/ActionController/RequestForgeryProtection/ClassMethods.html
-->

### 悪い例

変更が伴うアクションでCSRFトークン検証を省略すべきではありません。

```ruby
class HogeController < ApplicationController
  protect_from_forgery :except => [:index, :show, update]
```

```ruby
class HogeController < ApplicationController
  skip_forgery_protection
```

```ruby
class HogeController < ApplicationController
  skip_before_action :verify_authenticity_token
```

### 良い例

参照系のアクションだが、クエリ文字列が長くなってしまうためにHTTPメソッドをPOSTに変更しているような場合、CSRFトークン検証を省略しても安全です。

```ruby
class HogeController < ApplicationController
  protect_from_forgery :except => [:search]
```

セッションに基づかないリクエスト(API等)の場合、CSRFトークン検証を省略しても安全です。

```ruby
class HogeApiController < ApplicationController
  skip_forgery_protection
```

## セキュリティ機能を無効化していないこと

Rails標準のセキュリティ機能を無効化すべきではありません。

### レビュールール

`config/application.rb` の各種設定を確認し、設定値の妥当性を確認します。

### 注意が必要な例

#### CSRFトークン検証を無効化

```ruby
config.action_controller.allow_forgery_protection = false
```

#### デフォルトのセキュリティヘッダーを削除

```ruby
config.action_dispatch.default_headers.clear
```

デフォルト値やヘッダの意味はRailsガイドの[9 デフォルトのヘッダー](https://railsguides.jp/security.html#%E3%83%87%E3%83%95%E3%82%A9%E3%83%AB%E3%83%88%E3%81%AE%E3%83%98%E3%83%83%E3%83%80%E3%83%BC)を参照してください。

## Ransack を ransackable なしで使ってないこと 

便利すぎる機能は意図せず脆弱性となってしまうこともあります。ここでは `ransack` を使った検索機能を例に挙げます。

`ransack` は高機能な検索機能を実装するのに便利ですが、認可を誤まって非公開の属性で検索出来てしまうと、応答の差異から値を特定できてしまいます。

### レビュールール

Gemfileに `ransack` が含まれている場合、ソースコードを `ransack` で検索します。下記3つの条件を満たす場合、脆弱です。

1. {モデル}.`ransack` のパラメータがユーザ入力
2. モデルにパスワードやトークン等、非公開の属性が含まれている
3. `ransackable_attribute` `ransackable_associations` `ransortable_attributes` がモデルに定義されていない、または非公開の属性が含まれている

ちょっと複雑ですが、要は非公開の属性やアソシエーションで検索できたらNGということです。

### 悪い例

1. `ransack` のパラメータがユーザ入力

```ruby
# app/controllers/users_controller.rb
def search
  @q = User.ransack(params[:q])
```

2. モデルに非公開の属性が含まれている

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

どの属性が非公開かはアプリケーションの仕様によりますが、少なくとも `password` や `api_key` は公開してはいけない情報でしょう。

3. `ransackable_attribute` `ransackable_associations` `ransortable_attributes` がモデルに定義されていない、または非公開の属性が含まれている

User モデルに `ransackable_attribute` が定義されていない場合、脆弱です。

### 良い例

次の 1, 2, 3 いずれかに当てはまる場合、安全です。

1. `ransack` のパラメータがユーザ入力ではない

```ruby
Article.ransack(id_eq: 1)
```

2. モデルに非公開の属性が含まれていない

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

3. `ransackable_attribute` `ransackable_associations` `ransortable_attributes` がモデルに定義されている

```ruby
# app/models/user.rb
def self.ransackable_attributes(auth_object = nil)
  super & %w(email first_name last_name)
end

def self.ransortable_attributes(auth_object = nil)
  ransackable_attributes(auth_object)
end

def self.ransackable_associations(auth_object = nil)
  []
end
```

### 補足

`ransackable_` については [Ransack: Authorization (whitelisting/blacklisting)](https://github.com/activerecord-hackery/ransack#authorization-whitelistingblacklisting) を参照してください。