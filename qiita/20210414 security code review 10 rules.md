# 【Rails】コードレビューで脆弱性を発見しよう！セキュリティコードレビューのルール10選

コードレビューは誰かが書いたソースコードを確認することで、ソフトウェアの問題を発見する作業です。

コードレビューで発見できる問題には、コーディング規約違反や機能的なバグだけでなく、セキュリティの欠陥、つまり脆弱性も含まれます。

コードをマージする前に脆弱性を発見して潰すことができたらみんな幸せですね！

本記事では筆者が実際に脆弱性を発見したことがあるコードレビューのルールを10個紹介します。コードレビューの参考にしてください。

## 1. JSON からの情報漏洩

SPAなどAjaxによる画面更新を実装するために JSON を返すアクションを作ることがあります。その際、オブジェクトを不用意にシリアライズしてしまうと意図しない情報漏洩につながります。

### レビュールール

コントローラを正規表現 `render.+?json` で検索し、必要最小現の属性のみシリアライズされていることを確認します。

### 悪い例

オブジェクトをそのまま render

```ruby
render :json => @user
```

user に氏名住所など非公開情報や、パスワードのような秘密情報が含まれていたら大変ですね。

### 良い例

画面描写に必要な属性だけシリアライズ

```ruby
render :json => { :id => @user.id, :first_name => @user.first_name }
```

### 補足

jb や jbuilder を使っている場合は、テンプレートに不要な属性が含まれていないことを確認してください。

## 2. Strong Parameters が適切に設定されていること

Strong Parameters で更新可能なパラメータを不用意に permit してしまうと、開発者が意図しないデータを変更される危険性があります。

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

admin (管理者フラグ) を変更できてしまうのは危険です。

### 良い例

必要最小限の属性だけ permit

```ruby
params
.require(:user)
.permit(:email, :first_name, :last_name)
```

## 3. クエリに認可が含まれていること

クエリに認可が含まれていない場合、URL等に含まれるIDが改ざんされて他のユーザのリソースが不正に操作されてしまうことがあります。

例）自分のプロフィール画面のURL http://example.com/customers/123 の `123` を `444` に変えてみたら、他人のプロフィールが見えてしまった。

### レビュールール

コントローラを正規表現 `params\[:.*id\]` で検索し、リソースにアクセス制御が必要な場合、アクションまたはクエリにアクセス制御が含まれているかを確認します。

### 悪い例

全てのリソースを更新できてしまう

```ruby
def update
  @article = Articles.find(params[:article_id])
  @article.update!(article_params)
  redirect_to @article
end
```

`article_id` パラメータを変更することで、他人の article も更新できてしまいます。

ただし、下記の場合は安全です。

- コントローラ、アクションでアクセス制御を実装している（例えば管理者のみ使用できる、など）
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

`article_id` パラメータを変更しても、ユーザ自身の article しか参照・更新できないので安全です。

## 4. CORS が適切に構成されていること

CORSの設定ミスは Same Origin Policy によるセキュリティを緩めてしまい、意図しない情報漏洩につながる可能性があります。

### レビュールール

すべてのコードを `Access-Control-Allow-Origin` で検索し、信頼できない Origin を設定していないことを確認します。

### 悪い例

ワイルドカード

```ruby
headers['Access-Control-Allow-Origin'] = '*'
```

リクエストヘッダをコピー

```ruby
headers['Access-Control-Allow-Origin'] = request.headers["Origin"]
```

null

```ruby
headers['Access-Control-Allow-Origin'] = 'null'
```

※非公開情報を含まない、オープンなAPIの場合のみ `*` を設定しても安全です。

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

凝ったCORS設定が必要な場合は[`rack-cors`](https://github.com/cyu/rack-cors)を使う場合もありますが、その場合も `Access-Control-Allow-Origin` の妥当性は確認しましょう。

CORSについての詳細は[MDN: オリジン間リソース共有 (CORS)](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS)を参照してください。

## 5. レースコンディション：ファイルパスが重複しないこと

一時ファイルなどを固定のファイルパスに書き込みしている場合、同時に複数のリクエストが実行されるとデータが壊れてしまいます。これによって整合性がとれなくなったり、意図しない情報漏洩につながったしります。

### レビュールール

すべてのコードを正規表現 `File.open\(.+?"w` で検索し、ファイルパスが重複する可能性がないことを確認します。

### 悪い例

固定値

```ruby
File.open("tmp/upload-file", "w")
```

ユーザがアップロードしたファイルの名前

```ruby
file = fileupload_param[:file]
output_path = Rails.root.join('public', file.original_filename)
File.open(output_path, "w")
```

※該当するロジックがアクションから呼び出されない場合は安全です。

### 良い例

重複する可能性がないか超低い

```ruby
output_path = Rails.root.join('tmp', "#{SecureRandom.uuid}")
File.open(output_path, "w")
```

### 補足

そもそも一時ファイルを作るだけなら `tempfile` を使いましょう。

## 6. レースコンディション：時刻をIDにしていないこと

ファイルだけでなく、IDやキーも同様に重複するとレースコンディションが発生する可能性があります。

過去に見受けられたケースとして、時刻をIDとして使ってしまうケースがありました。利用者が少数の時は発現する可能性は低いですが、利用者が増えてくるとリクエストが同時に処理される可能性が高くなり、複数のリクエスト間で同一のリソースを操作してしまいます。これによってデータの整合性がとれなくなったり、意図しない情報漏洩につながったしります。

### レビュールール

すべてのコードを正規表現 `Time\.(now|current|zone\.now)` で検索し、IDやファイル名などを現在時刻だけで作成していないかを確認します。

### 悪い例

IDが時刻のみで構成されている

```ruby
File.open("tmp/#{Time.zone.now.to_i}", "w")
```

※概要するロジックがアクションから呼び出されない場合は安全です

### 良い例

IDが時刻＋ユニークな値で構成されている

```ruby
File.open("tmp/#{Time.zone.now.to_i}_#{SecureRandom.uuid}", "w")
```

### 補足

IDとなりうるものとして、ファイルやデータベースのレコードだけでなく、S3のオブジェクトキーなども含まれます。

## 7. CSRF対策（GETメソッドでリソースを変更していないこと）

GETメソッドのアクションでリソースの状態を変更すると、CSRF攻撃に対して脆弱になります。

### レビュールール

GETメソッドでリソースを変更していないことを確認します。どのアクションがGETメソッドなのかは `rails routes` の出力を見ると良いでしょう。

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

## 8. CSRF対策（トークン検証を省略していないこと）

CSRFトークンの検証を安易に無効化すべきではありません。

過去の事例として、Ajax通信をすると `Can't verify CSRF token authenticity` エラーが出るからと言って検証を無効化しているケースがありました。

### レビュールール

コントローラを次の文字列で検索し、トークン検証を無効化していないことを確認します。

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
  protect_from_forgery :except => [:index, :show, :update]
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

参照系のアクションだけど、クエリ文字列が長くなってしまうためにHTTPメソッドをPOSTに変更しているような場合、CSRFトークン検証を省略しても安全です。

```ruby
class HogeController < ApplicationController
  protect_from_forgery :except => [:search]
```

セッションに基づかないリクエスト(API等)の場合、CSRFトークン検証を省略しても安全です。

```ruby
class HogeApiController < ApplicationController
  skip_forgery_protection
```

### 補足

Rails の CSRF 対策については、[Rails セキュリティガイド:CSRFへの対応策](https://railsguides.jp/security.html#csrf%E3%81%B8%E3%81%AE%E5%AF%BE%E5%BF%9C%E7%AD%96) を参照してください。


## 9. 標準のセキュリティ機能を無効化していないこと

Rails 標準のセキュリティ機能を無効化すべきではありません。

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

## 10. Ransack を ransackable なしで使ってないこと 

便利すぎる機能は、時として脆弱性となってしまうこともあります。ここでは `ransack` を使った検索機能を例に選びました。

`ransack` は高機能な検索機能を実装するのに便利ですが、非公開の属性で検索出来てしまうと、検索結果の差異から非公開の値を特定できてしまいます。これは暗号化されたパスワードやトークンなど、データベースに保存された秘密情報の漏洩につながります。

### レビュールール

Gemfileに `ransack` が含まれている場合、ソースコードを `ransack` で検索します。下記3つの条件を満たす場合、脆弱です。

1. {モデル}.`ransack` のパラメータがユーザ入力
2. モデルにパスワードやトークン等、非公開の属性が含まれている
3. `ransackable_attribute`, `ransackable_associations`, `ransortable_attributes` がモデルに定義されていない、または非公開の属性が含まれている

ちょっと複雑ですが、要は非公開の属性やアソシエーションで検索できたらNGということです。

### 悪い例

1\. `ransack` のパラメータがユーザ入力

```ruby
# app/controllers/users_controller.rb
def search
  @q = User.ransack(params[:q])
```

2\. モデルに非公開の属性が含まれている

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

どの属性が非公開かはアプリケーションの仕様によりますが、少なくとも `password` や `api_key` は公開してはいけない属性でしょう。

3\. `ransackable_attribute`, `ransackable_associations`, `ransortable_attributes` がモデルに定義されていない、または非公開の属性が含まれている

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # def ransackable_* がない
end
```

### 良い例

次の 1, 2, 3 いずれかに当てはまる場合、安全です。

1\. `ransack` のパラメータがユーザ入力ではない

```ruby
Article.ransack(id_eq: 1)
```

2\. モデルに非公開の属性やアソシエーションが含まれていない

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

3\. `ransackable_attribute`, `ransackable_associations`, `ransortable_attributes` がモデルに定義されており、必要な属性・アソシエーションに絞られている

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # ...

  def self.ransackable_attributes(auth_object = nil)
    super & %w(email first_name last_name)
  end

  def self.ransortable_attributes(auth_object = nil)
    ransackable_attributes(auth_object)
  end

  def self.ransackable_associations(auth_object = nil)
    []
  end

  # ...
end
```

### 補足

ransack の認可 `ransackable` の詳細は [Ransack: Authorization (whitelisting/blacklisting)](https://github.com/activerecord-hackery/ransack#authorization-whitelistingblacklisting) を参照してください。

`ransack` で検索してもマッチしない場合、`search` を使っている可能性があります。詳細は [Ransack #search method](https://github.com/activerecord-hackery/ransack#ransack-search-method) を参照してください。

## Next step

脆弱性を発見するコードレビュールールを10個紹介しました。こんな感じでコードレビューをすると脆弱性を発見できるかもしれません。

次に、よりセキュアにアプリケーション開発をするためにできることを書いておきます。

### Brakeman

今回はソースコードを目視で確認しましたが、自動で解析してくれるツールもあります。

[Brakeman](https://brakemanscanner.org/) は、SQLインジェクションやクロスサイトスクリプティングなどインジェクション系の脆弱性やセキュリティ設定の不備をそれなりの精度で検出してくれる便利なツールです。

ただし Brakeman は全ての脆弱性を見つけてくれるわけではありません。本記事で紹介した脆弱性のほとんどは Brakeman では検出できません。

そのため、実際のコードレビューでは Brakeman と目視によるレビューの両方で相互補完するのが効果的です。

### Ruby on Rails セキュリティガイド

脆弱性がコードレビューで見つかるのはとても良いことですが、そもそも作りこまないことも重要です。脆弱性を作りこみにくい、安全にコーディングすることを**セキュアコーディング**といいます。

[Rails セキュリティガイド](https://railsguides.jp/security.html) にはWebアプリケーションの脆弱性とRailsにおける対策方法が満載です。セキュアコーディングの実践に大いに役に立つでしょう。

### セキュリティコードレビュールールの改善

本記事で紹介したルールはあくまで例なので、アプリケーションによっては検索にマッチしすぎたり、逆に見つけられなかったりすることもあります。

[Rails セキュリティガイド](https://railsguides.jp/security.html)を参考にしたり、過去に発覚した脆弱性をもとにルールを追加・変更していきましょう。

本記事のルールを改善する良いアイデアがあったらぜひ教えてください！
