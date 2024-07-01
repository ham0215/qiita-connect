---
title: graphql-rubyを使って認可する方法
tags:
  - Ruby
  - Rails
  - GraphQL
  - 認可
  - graphql-ruby
private: false
updated_at: '2022-04-19T12:34:18+09:00'
id: fa4ce59b5cbedfb1fb7d
organization_url_name: null
slide: false
ignorePublish: false
---
GraphQLを使っているときに様々な処理で認可させたい事があると思います。

* このQueryはログインユーザーのみ実行できるようにしたい
* このMutationは管理者のみ実行できるようにしたい
* このQueryは自分の所有しているデータのときだけ返却するようにしたい
* このFieldは自分の所有しているデータのときだけ返却するようにしたい

当初はgraphql-rubyの知識が乏しかったので取得や更新処理の中で認可する処理を呼び出していたのですが、graphql-rubyのドキュメントを改めて読み直したところ、認可のためのメソッド(authorized?)がある事がわかったので動作検証を兼ねて記事を書きました。

2021/02/10 追記

コメントでいただきましたが、類似のメソッドに`ready?`というものがあるようです。
ドキュメントにも記載されています。
https://graphql-ruby.org/mutations/mutation_authorization.html

ready?とauthorized?の違いはargumentsがロードされる前に呼ばれるか否かです。
ready?はargumentsがロードされる前に呼ばれ、authorized?はロード後に呼ばれるようです。

そのため、argumentsを使わないチェックの場合はready?で行った方が効率が良さそうです。

# graphql-rubyについて

Ruby(Rails)でGraphQLを簡単に使えるようにしてくれるGemです。
https://github.com/rmosolgo/graphql-ruby

細かいところは実際に試してみないとわからないことも多いですが、ドキュメントが充実していて素晴らしいです。
https://graphql-ruby.org/guides

この記事を書いている時点では、`graphql: 1.11.1`を使っています。
まだガンガンバージョンアップしているGemなので、バージョンが違うと大幅に動作が変わっている可能性があるのでご注意ください。

# 認可の実装例

最初に挙げた4つのパターンの実装例を説明します。

## 前提条件

認可に必要なログインユーザーの情報はcontextに格納していることとします。
認証についてはこの記事の本筋からの逸れるので説明は省略します。

```ruby:app/controllers/graphql_controller.rb
# ログインユーザーの情報はcontext[:current_user]に格納
# 未ログインの場合はnil
context = { current_user: current_user }
```

## このQueryはログインユーザーのみ実行できるようにしたい

ここでは『review_idを指定して該当するReviewTypeを返却するクエリー』を実装します。

### 認可を入れる前

認可を実装する前にReviewTypeを取得するクエリーを実装します。

```ruby:app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    field :review, resolver: Resolvers::ReviewResolver
  end
end
```

```ruby:app/graphql/resolvers/review_resolver.rb
module Resolvers
  class ReviewResolver < BaseResolver
    type Types::ReviewType, null: true

    argument :review_id, Int, required: true

    def resolve(review_id:)
      Review.find_by(id: review_id)
    end
  end
end
```

```ruby:app/graphql/types/review_type.rb
module Types
  class ReviewType < BaseObject
    field :id, ID, null: false
    field :title, String, null: true
    field :body, String, null: true
    field :secret, String, null: true
    field :user, Types::UserType, null: false
  end
end
```

```ruby:app/graphql/types/user_type.rb
module Types
  class UserType < BaseObject
    field :id, ID, null: false
    field :name, String, null: false
    field :email, String, null: false
  end
end
```

GraphiQLで実行すると次のようになります。
<img width="562" alt="スクリーンショット 2020-07-24 13.51.33.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/83424/1e2aaaa9-eea8-85a7-db71-21b5d164733e.png">

### 認可を実装
それでは先ほど実装した処理に『ログインユーザーのみ実行できる』という制約を追加します。

#### authorized?を使わない実装

以前の私はresolveメソッドでReviewを取得する前にログインチェックする実装を入れていました。

まずは様々なResolverから使えるようにBaseResolverにログインチェックメソッドを実装します。
context[:current_user]が入っていない場合はエラーを発生させます。
ちなみに、`GraphQL::ExecutionError`を使うとraiseするだけでレスポンスをGraphQLのエラー形式に変換してくれます。

```ruby:app/graphql/resolvers/base_resolver.rb
 module Resolvers
   class BaseResolver < GraphQL::Schema::Resolver
     def login_required!
       # ログインしていなかったらraise
       raise GraphQL::ExecutionError, 'login required!!' unless context[:current_user]
     end
   end
 end
```

次にBaseResolverのログインチェックを処理の最初に呼び出すようにします。

```diff:app/graphql/resolvers/review_resolver.rb
 def resolve(review_id:)
+  # 処理の最初にログインチェックを行う
+  login_required!

   Review.find_by(id: review_id)
 end
```

GraphiQLで未ログインの状態で実行すると次のようになります。
<img width="641" alt="スクリーンショット 2020-07-25 12.53.22.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/83424/c6c4c4c7-d3dd-b945-a25c-ab275a989036.png">

この方法でもやりたいことは実現できているのですが、ログイン必須のResolverは処理の最初に必ず`login_required!`を書かなければいけません。
controllerのbefore_actionのように本処理が呼ばれる前に自動で認可してくれる方法はないのかをずっと探していました。

#### authorized?を使う実装

graphql-rubyのガイドを改めて読んでいるとauthorized?というメソッドがあることに気づきました。
これを使うとresolveメソッドの前に認可を行い、実行可否を制御することができるようです。
下記はmutationに追加するガイドですが、Resolverにも同じように追加できます。
https://graphql-ruby.org/mutations/mutation_authorization.html

ログイン必須のResolverは汎用的に使えそうなので、ログイン必須のResolverが継承するlogin_required_resolverを作りました。
authorized?のパラメーター(args)にはresolveと同じパラメーターが格納されます。

```ruby:app/graphql/resolvers/login_required_resolver.rb
module Resolvers
  class LoginRequiredResolver < BaseResolver
    def authorized?(args)
      context[:current_user].present?
    end
  end
end
```

review_resolverはlogin_required_resolverを継承するように修正します。
他の実装は認可を追加する前と同じです。

```diff:app/graphql/resolvers/review_resolver.rb
- class ReviewResolver < BaseResolver
+ class ReviewResolver < LoginRequiredResolver
```

GraphiQLで未ログインの状態で実行すると次のようになります。
<img width="524" alt="スクリーンショット 2020-07-25 13.01.45.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/83424/bb3c29a4-265c-63be-8475-2bd062ed6424.png">

authorized?の結果がfalseの場合はエラー情報はなく`data: null`だけ返却されるようになりました。
ガイドにも記載がある通り、authorized?がfalseの場合は`data: null`だけを返却するのがデフォルトの挙動のようです。
nullを返却するという仕様で問題なければこのままで良いですが、認可されない場合はエラー情報も返却するように変更してみます。

エラー情報を追加する方法は簡単で、authorized?の中でGraphQL::ExecutionErrorをraiseすればできます。
ちなみに成功時はtrueを明示的に返却しないと成功と認識されないので注意が必要です。

```ruby:app/graphql/resolvers/login_required_resolver.rb
module Resolvers
  class LoginRequiredResolver < BaseResolver
    def authorized?(args)
      # 認可できない場合はGraphQL::ExecutionErrorをraise
      raise GraphQL::ExecutionError, 'login required!!' unless context[:current_user]

      true
    end
  end
end
```

GraphiQLで未ログインの状態で実行すると次のようになります。
これでauthorized?を使った場合でもエラー情報を返却することができました。
<img width="645" alt="スクリーンショット 2020-07-25 13.25.54.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/83424/9a37940b-a5fe-9706-2deb-f534ea1a4e6d.png">

authorized?を使った場合、resolveメソッドでは認可の処理を書く必要がなくなるのでシンプルに書くことができます。
（今回の例はかなりシンプルな実装なのでそこまで差はありませんが・・・）

## このMutationは管理者のみ実行できるようにしたい

ここでは『review_idを指定して該当するReviewのtitleとbodyを更新するMutation』を実装します。

### 認可を入れる前に

認可を実装する前にReviewを更新するMutationを実装します。
１つ前の例で使ったReviewTypeなどそのまま使うクラスは省略します。

```ruby:app/graphql/types/mutation_type.rb
module Types
  class MutationType < Types::BaseObject
    field :update_review, mutation: Mutations::UpdateReview
  end
end
```

```ruby:app/graphql/mutations/update_review.rb
module Mutations
  class UpdateReview < BaseMutation
    argument :review_id, Int, required: true
    argument :title, String, required: false
    argument :body, String, required: false

    type Types::ReviewType

    def resolve(review_id:, title: nil, body: nil)
      review = Review.find review_id
      review.title = title if title
      review.body = body if body
      review.save!

      review
    end
  end
end
```

GraphiQLで実行すると次のようになり、Reviewデータが更新されます。
<img width="820" alt="スクリーンショット 2020-07-27 11.34.17.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/83424/6cf0d263-3773-02c0-d92c-ae3db4a46822.png">

### 認可を実装
Mutationでも先程の例と同様にauthorized?を使うことができます。
下記のガイトに記載されています。
https://graphql-ruby.org/mutations/mutation_authorization.html

管理者しか利用できないMutationが継承する親クラスを作って継承するようにします。

```ruby:app/graphql/mutations/base_admin_mutation.rb
module Mutations
  class BaseAdminMutation < BaseMutation
    def authorized?(args)
      raise GraphQL::ExecutionError, 'login required!!' unless context[:current_user]
      raise GraphQL::ExecutionError, 'permission denied!!' unless context[:current_user].admin?

      super
    end
  end
end
```

```diff:app/graphql/mutations/update_review.rb
- class UpdateReview < BaseMutation
+ class UpdateReview < BaseAdminMutation
```

Mutationのauthorized?もfalseを返却するだけだとエラー情報は返却されず、dataがnullになり更新処理が実行されないようになります。
Resolverはそれでも良さそうですがMutationはエラー情報を返却しないとよくわからないと思うので、こちらもGraphQL::ExecutionErrorをraiseするように実装しました。
ちなみにガイドを読むと下記のように戻り値にerrorsを返却することでエラー情報を返す方法もあるようです。
試してみましたが下記の方法ではerrors配下のlocationsやpathは返却されませんでしたが、errorsのmessageは返却できました。
メッセージだけ返却できればよいのであればどちらの方法で実装しても良さそうです。

```ruby
def authorized?(employee:)
  if context[:current_user]&.admin?
    true
  else
    return false, { errors: ["permission denied!!"] }
  end
end
```

GraphiQLで管理者権限を持っていないユーザーが実行すると次のようになります。
もちろんエラーの場合は更新処理は実行されません。
<img width="958" alt="スクリーンショット 2020-07-27 11.48.29.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/83424/2f521f46-b581-f057-9c92-7a390814ceba.png">

## このQueryは自分の所有しているデータのときだけ返却するようにしたい

ここでは最初に作った『review_idを指定して該当するReviewTypeを返却するクエリー』を基に改修します。
最初に作ったものはログイン状態のみ確認していましたが、今回はReviewが自分の所有物か？のチェックを追加します。

### ログインチェックと同じauthorized?に実装してみる

ログインチェックと同じauthorized?にチェックを追加できればよいのですが、今回のチェックはRevewを取得した後でないとチェックできません。
authorized?でもreview_idは引数で受け取るのでReviewを取得することもできるのですが、そうするとresolveの役割が曖昧になります。
実際に実装してみます。

```diff:app/graphql/resolvers/login_required_resolver.rb
 def authorized?(args)
   raise GraphQL::ExecutionError, 'login required!!' if context[:current_user].blank?

+   # この時点でreviewの取得が必要
+   review = Review.find_by(id: args[:review_id])
+   return false unless review
+   raise GraphQL::ExecutionError, 'permission denied!!' if context[:current_user].id != review.user_id

   true
 end
```

authorized?でReviewの取得が必要になります。
resolveメソッドでも取得するので、ここでも取得すると非効率な気がしますね。
では、resolve側にチェックを実装するとどうでしょうか？

```diff:app/graphql/resolvers/review_resolver.rb
 def resolve(review_id:)
-   Review.find_by(id: review_id)
+   review = Review.find_by(id: review_id)
+   raise GraphQL::ExecutionError, 'permission denied!!' if context[:current_user].id != review.user_id

+   review
 end
```

こちらの方がauthorized?で実装するより効率は良さそうですが、authorized?にチェック処理を切り出すことでデータ取得処理のみ記載していたresolveにまたチェック処理が入ってしまいました。

当初はデータ取得後にしかチェックできないものがresolveでチェックするしかないと思っていたのですが、authorized?はReviewTypeにも定義できることを知ったのでReviewTypeに定義してみます。

### ReviewTypeでチェックする

ReviewTypeでチェックするとはどういうことなのか？
実際に実装してみます。

ReviewTypeは誰でも使えるようにしておきたいので、MyReviewTypeという自分しか閲覧できない制約をつけたReviewTypeを作ります。

```ruby:app/graphql/types/my_review_type.rb
module Types
  class MyReviewType < ReviewType
    def self.authorized?(object, context)
      raise GraphQL::ExecutionError, 'permission denied!!' if context[:current_user].id != object.user_id

      true
    end
  end
end
```

ガイドにも記載されていますが、Typeで使うauthorized?はobjectとcontextを引数に受け取ります。
あと、クラスメソッドなので注意が必要です。
https://graphql-ruby.org/authorization/authorization.html

あとはレスポンスのTypeをMyReviewTypeにするだけです。他の修正は不要です。

```diff:app/graphql/resolvers/review_resolver.rb
- type Types::ReviewType, null: true
+ type Types::MyReviewType, null: true
```

GraphiQLで自分以外のReviewを指定すると次のようになります。
<img width="835" alt="スクリーンショット 2020-07-27 22.40.15.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/83424/56883594-2157-117b-3fe9-3b2a13964944.png">

これでresolveメソッドには認可の処理を書く必要がなくなるのでシンプルに書くことができました。
また、レスポンスをMyReviewTypeにすることでスキーマ定義を読むだけで、このクエリーはMyReviewTypeを返却する＝「自分しか閲覧できない」ということが明確になるので良いと思います。

## このFieldはログインユーザー自身のデータのときだけ返却する

1つ前の例ではMyReviewTypeを定義してレスポンス全体を自分のデータのときしか見れないようにしました。
しかし、全部ではなく特定のフィールドだけ見れないようにしたいこともあると思います。

ReviewTypeを再掲します。
ここではsecretカラムは自分のデータしか見れないようにしたいと思います。

```ruby:app/graphql/types/review_type.rb
module Types
  class ReviewType < BaseObject
    field :id, ID, null: false
    field :title, String, null: true
    field :body, String, null: true
    field :secret, String, null: true # <- これを自分の場合のみ見えるようにする
    field :user, Types::UserType, null: false
  end
end
```

ガイドを読むとfieldにもauthorized?が実装できるようなのですが、1つのfieldだけをカスタマイズするのは難しそうなのでここではauthorized?を使わずに実装することにしました。
https://graphql-ruby.org/authorization/authorization.html
fieldのガイドはこちら
https://graphql-ruby.org/fields/introduction.html#field-parameter-default-values

下記のようにfield名と同じメソッドを定義すると、そちらが呼び出されるようになります。
そのメソッド内に認可を実装しました。

```ruby:app/graphql/types/review_type.rb
module Types
  class ReviewType < BaseObject
    field :id, ID, null: false
    field :title, String, null: true
    field :body, String, null: true
    field :secret, String, null: true
    field :user, Types::UserType, null: false

    # field名のメソッドを定義すると呼び出される
    def secret
      # ログインユーザーとレビューを書いたユーザーが違う場合、nilを返却
      return if object.user_id != context[:current_user].id

      object.secret
    end
  end
end
```

GraphiQLで自分以外のReviewを指定すると次のようになります。
secretはnullが返却されています。
<img width="809" alt="スクリーンショット 2020-07-28 22.51.28.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/83424/2b2130ec-52a9-e996-a5bd-bd8f0d33c544.png">

Resolverにこのチェックを実装するとReviewTypeを使うすべてのResolverがsecretの考慮をしなければいけなくなりますが、ReviewTypeに実装することで個別のResolverはsecretのアクセス制御を考える必要がなくなります。

# 最後に

graphql-rubyを使い始める前にもガイドは一通り目を通したつもりだったのですが、authorized?の存在は見落としていました・・・
authorized?以外にもまだまだ気づいていない便利な機能がありそうですね。
また、今はなかったとしてもがんがんバージョンアップされており、これからも新しい機能が追加される可能性も高いので、これからもgraphql-rubyの動向をチェックしていきたいと思います。
