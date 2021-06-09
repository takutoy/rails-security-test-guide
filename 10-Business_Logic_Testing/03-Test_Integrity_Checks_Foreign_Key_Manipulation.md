# 4.10.3 Test Integrity Checks - 外部キーの不正な更新

## 概要

あるレコードの外部キーの値を変更できる場合、権限を持たないリソースに関連付けできてしまう場合があります。これはデータの破壊や情報漏洩につながる可能性があります。

このテストでは、ユーザから入力された外部キーの値が検証され、不正なデータの関連付けから保護されていることを確認します。

## 脆弱性の原理

次のようなデータ構造を持ったイシュー管理SaaSを考えます。

- ユーザ登録されると organization が1つ作成されます
- organization は、1人以上の user を持ちます
- organization は、0個以上の issue を持ちます
- issue には organization に所属する user を割り当てることができます


<!--
```plantuml
@startuml
entity "Organization" as e01 {
  *id : integer
  --
  name : string
}

entity "User" as e02 {
  *id : integer
  *org_id : integer <<FK>>
  --
  name : string
}

entity "Issue" as e03 {
  *id : integer
  *org_id : integer <<FK>>
  *user_id : integer <<FK>>
  --
  title : string
}

e01 ||..o{ e02
e01 ||..o{ e03
e02 ||..o{ e03
@enduml
```
-->

![](images/2021-06-09-21-27-39.png)

ユーザは issue の担当者を割り当てるために、issue の外部キーである user_id を変更することができます。

もしユーザが user_id を任意の値で更新できるとしたら、他の organization に所属している user に issue を割り当てることができてしまいます。これはデータの不整合を発生させるだけではなく、情報漏洩を引き起こす可能性もあります。

<!--

例えば次のようなコントローラとビューがある場合、他の organization に所属している user の名前やメールアドレスが参照できてしまいます。

```ruby
# app/controllers/issues_controller.rb
def index
    @issues = @organization.issues.includes(:user)
end
```

```html
<!-- app/views/issues/index.html.erb
<tbody>
  <% @issues.each do |issue| %>
  <tr>
    <td><%= issue.title %></td>
    <td><%= issue.user.name %></td>
  </tr>
  <% end %>
</tbody>
```
-->

次のようなデータで考えてみましょう。

![](images/2021-06-09-21-30-20.png)

Foo Co. に所属する suzuki taro でログインすると、次のような画面が表示され、Issue に自組織のユーザをアサインすることができます。

![](images/2021-06-09-21-31-23.png)

ここで、もしアサインする担当者の user_id が自組織に所属するユーザのIDであることが検証されていない場合、次のような攻撃が可能になります。

![](images/2021-06-09-21-32-12.png)

Foo Co. に所属する suzuki taro が、他組織である Hoge Inc. のユーザの名前を参照できてしまいました。これは情報漏洩です！

## 静的テスト

外部キーを更新しているコードとその周辺をレビューします。

もし外部キーが参照するリソースがアクセス制御を必要とする場合、更新しようとしている外部キーの値は、ユーザがアクセスできるリソースのID、であることを検証する必要があります。

SaaS型イシュー管理アプリケーションの場合、ユーザは他の organization に所属する user にアクセスできてはいけないため、issue 更新リクエストの user_id パラメータを検証する必要があります。

### 外部キーを更新するコードの例

update の属性に `xxx_id` がある

```ruby
issue.update(user_id: params[:user_id])
```

String Parameters の permit に `xxx_id` がある

```ruby
params.require(:issue).permit(:title, :description, :user_id)
```

### 外部キーを検証するコードの例

update が実行される前に、`user_id` パラメータが検証されているかを確認します。

検証がない場合、脆弱な可能性があります。

もし次のように `user_id` が検証されており、かつロジックの妥当性が確認できれば安全であると判断できます。

```ruby
unless @organization.users.exists?(id: params[:issue][:user_id])
  return head 403
end
```

※検証ロジックの有効性を確認するため、動的テストも実施することを推奨します

## 動的テスト

リソースの作成・更新リクエストを改ざんし、外部キー(`xxx_id`)の値を変更して送ります。

リクエストの例

![](images/2021-06-03-23-58-27.png)

`user_id` を改ざんしたリクエスト

![](images/2021-06-03-23-58-57.png)

外部キーの更新に成功した場合、脆弱であると判断します。