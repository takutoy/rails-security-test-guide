# 変数のスコープの誤りによる競合や漏洩 - Race Condition and Information Disclosure by Improper Variable Scope

## 概要

変数のスコープが不適切に実装された場合、同時アクセスによるデータ競合や漏洩が発生する可能性があります。

変数に次のスコープが使用されている場合に注意が必要です。

- 競合
  - グローバル変数 (`$`)
  - クラス変数 (`@@`)
  - クラスインスタンス変数 (クラスメソッド内で宣言された`@`) ※
- 漏洩
  - スレッドローカル変数 (`Thread#[]=`)
  - per-thread クラス変数 (`thread_mattr_accessor`)

※通常のインスタンス変数は安全です

<!--
検索キーワードの例：
`\$[_a-zA-Z].*[^!=]=|@@|thread_[cm]attr|Thread.current`
-->

## 脆弱なコードの例(競合)

複数のリクエスト間で変数が共有(変更を伴う)される場合、競合が発生する可能性があります。（問題が発生するのは変数が"変更"される場合であり、"参照"だけであれば問題は発生しません。）

なお、下記のコードでは分かりやすさのためにコントローラにグローバル変数やクラス変数を定義していますが、モデルに定義する場合でも同様の問題が発生します。

### グローバル変数

次のコードを見てみます。

```ruby
class HomeController < ApplicationController
    def index
        $current_user_global = User.find(session[:user_id])
        sleep 1 # なんかいろんな処理
        render plain: "session[:user_id]=#{session[:user_id]}" + 
                      "$current_user_global.id=#{$current_user_global.id}"
    end
end
```

一見すると `session[:user_id]` と `$current_user_global.id` は同じ値になると期待します。

```
session[:user_id]=1, $current_user_global.id=1
```

しかしグローバル変数 `$current_user_global` は別のリクエストとも共有されるため、リクエスト1の実行中にリクエスト2が発生した場合に上書きされてしまい、次のような結果になることがあります。

```
session[:user_id]=1, $current_user_global.id=2
```

上記のコードの場合、ログインしているユーザとは別のユーザとして処理が実行されることになってしまいます。

### クラス変数

クラス変数を使う場合も同様です。

```ruby
class HomeController < ApplicationController
    def index
        @@current_user_class = User.find(session[:user_id])
        sleep 1 # なんかいろんな処理
        render plain: "session[:user_id]=#{session[:user_id]}, " +
                      "@@current_user_class.id=#{@@current_user_class.id}"
    end
end
```

### クラスインスタンス変数

クラスインスタンス変数も同様です。

```ruby
class HomeController < ApplicationController
    def index
        test = HomeController.set_current_user(session[:user_id])
        render plain: test
    end

    def self.set_current_user(user_id)
        # クラスインスタンス変数（インスタンス変数ではない）
        @current_user_class_instance = User.find(user_id)
        sleep 1 # なんかいろんな処理
        "user_id=#{user_id}, " + 
        "@current_user_class_instance.id=#{@current_user_class_instance.id}"
    end
end
```

## 脆弱なコードの例(情報漏洩)

スレッドローカル変数を適切に初期化していない場合、前のリクエストのデータが残留し、意図せず情報を漏洩してしまったり不正な処理を実行してしまう可能性があります。

※スレッドローカル変数による問題は、Webサーバのスレッド制御方法や設定によっては発現しない場合もあります。

### スレッドローカル変数

```ruby
class HomeController < ApplicationController
    def index
        test = "session[:user_id]=#{session[:user_id]}, " + 
               "Thread.current[:user]&.id=#{Thread.current[:user]&.id}"
        Thread.current[:user] = User.find(session[:user_id])
        render plain: test
    end
end
```

このアクションに何度かアクセスすると、`Thread.current[:user]` に前のリクエストのデータが残ってることが確認できます。

```
session[:user_id]=1, Thread.current[:user]&.id=2
```

### thread_mattr_accessor

`thread_mattr_accessor` も内部で `Thread.current#[]` を使用しているため、同様の問題が発生します。

※要確認：Rails 7以降は安全かも。see https://github.com/rails/rails/commit/2bf5974287c218502ee56d5bfaaea973c351e47d

```ruby
class HomeController < ApplicationController
    thread_mattr_accessor :current_user_thread_local_accessor

    def index
        test = "session:#{session[:user_id]}, " +
               "current:#{current_user_thread_local_accessor&.id}"
        HomeController.current_user_thread_local_accessor = User.find(session[:user_id])
        render plain: test
    end
end
```

## 動的テスト

上記のようなアクションがある場合、複数のセッションで同時にアクセスし、単一でアクセスしたときと挙動に変化がないか観察します。

## 参考リンク

- [Ruby リファレンス：変数と定数](https://docs.ruby-lang.org/ja/latest/doc/spec=2fvariables.html)
- [Ruby on Rails API: thread_mattr_accessor](https://api.rubyonrails.org/classes/Module.html#method-i-thread_mattr_accessor)
- [Official Ruby FAQ: What is a class instance variable?](https://www.ruby-lang.org/en/documentation/faq/8/#:~:text=What%20is%20a%20class%20instance%20variable)
- [Redmine User.current からの漏洩の修正パッチ](https://www.redmine.org/issues/16685)