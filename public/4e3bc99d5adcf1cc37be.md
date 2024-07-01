---
title: '[Rails]アソシエーションのキャッシュ制御を活用して無駄なSQLを抑止する'
tags:
  - Ruby
  - Rails
private: false
updated_at: '2022-04-19T12:37:19+09:00'
id: 4e3bc99d5adcf1cc37be
organization_url_name: null
slide: false
ignorePublish: false
---
Active Recordには関連付け機能(アソシエーション)という強力な機能があります。
この記事ではアソシエーションのキャッシュ制御を活用して無駄なSQLを抑止する方法をまとめました。

# アソシエーションのキャッシュ制御とは
Railsガイドの下記のように記載されています。
[Active Record の関連付け- 3.1 キャッシュ制御](https://railsguides.jp/association_basics.html#%E3%82%AD%E3%83%A3%E3%83%83%E3%82%B7%E3%83%A5%E5%88%B6%E5%BE%A1)
> 最後に実行したクエリの結果はキャッシュに保持され、次回以降の操作で利用できます。

実際に動かしてキャッシュされているか確認します。
下記のモデルを使います。

```ruby
class User < ApplicationRecord
  has_many :reviews
end

class Review < ApplicationRecord
  belongs_to :user
end
```

下記のように`user.reviews`を実行すると、2回目の実行ではSQLが発行されていません。
これは1回目の実行結果をActiveRecordがキャッシュしており、それを返却しているためです。

```irb
irb(main):004:0> user = User.first
  User Load (5.8ms)  SELECT `users`.* FROM `users` ORDER BY `users`.`id` ASC LIMIT 1
=> #<User:0x000056307ae3aca0

irb(main):005:0> user.reviews
  Review Load (2.7ms)  SELECT `reviews`.* FROM `reviews` WHERE `reviews`.`user_id` = 1
=> [#<Review:0x000056307b1a77d0

irb(main):007:0> user.reviews
=> [#<Review:0x000056307b1a77d0
```

今回の例ではreviewsを取得するSQLの実行には2.7msかかっていることがわかります。
2回目はキャッシュを使っているため、同じ処理でも2.7ms節約できていることになります。

今回のように1つのSQL発行が短縮されただけだとあまり恩恵はありませんが、
これが100回、1,000回と積み重なると数秒の差となり、体感でも違いがわかるようになってきます。


# 取得のとき
先ほどの例と同じモデルを使います。
userオブジェクトを取得済みのとき、そのユーザーが持っているreviewsを取得する時はどのように取得しますか？

```ruby
# Userオブジェクト取得済み
user

# userの持っているreviewsを取得する
# 1
reviews = Review.where(user_id: user.id)

# 2
reviews = Review.where(user: user)

# 3
reviews = user.reviews
```

1~3はreviewsを取得する時に発行されるSQLは全部同じですが、１点違いがあります。
違いはreviewの関連データ`user`がキャッシュされているかどうかです。
irbで上記1~3の方法でreviewsを取得して、reviewsの関連データ`user`がキャッシュされているかどうか確認してみます。

```irb:#1
irb(main):009:0> reviews = Review.where(user_id: user.id)
  Review Load (1.2ms)  SELECT `reviews`.* FROM `reviews` WHERE `reviews`.`user_id` = 1
=> [#<Review:0x000055fc149bca10

irb(main):010:0> reviews.first.user
  User Load (0.7ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 1 LIMIT 1
=> #<User:0x000055fc14b75d20
```

```irb:#2
irb(main):011:0> reviews = Review.where(user: user)
  Review Load (0.8ms)  SELECT `reviews`.* FROM `reviews` WHERE `reviews`.`user_id` = 1
=> [#<Review:0x000055fc14b83ab0

irb(main):012:0> reviews.first.user
  User Load (0.7ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 1 LIMIT 1
=> #<User:0x000055fc14fca880
```

```irb:#3
irb(main):015:0> reviews = user.reviews
  Review Load (0.6ms)  SELECT `reviews`.* FROM `reviews` WHERE `reviews`.`user_id` = 1
=> [#<Review:0x000055fc1504c0b0

irb(main):016:0> reviews.first.user
=> #<User:0x000055fc14873f78
```

3の場合のみ`reviews.first.user`を実行したときにuserを取得するSQLが発行されていません。
アソシエーションで取得した場合は元となった関連データはキャッシュされるようです。

実際に動作確認する前は、2のようにwhereにオブジェクトを渡す方法もオブジェクトを渡しているのだから関連データがキャッシュされるのではないかと期待していたのですが、そのようには実装されていないようです。

関連データを取得する場合、アソシエーションを使って取得した方がキャッシュが活用できるので積極的に使っていきましょう。

# 作成のとき
先ほどの例と同じモデルを使います。
userオブジェクトを取得済みのとき、そのユーザーにreviewを追加する場合、どのように作成しますか？

```ruby
# Userオブジェクト取得済み
user

# userのreviewを新規作成する
# 1
review = Review.create!(content: 'hogehoge', user_id: user.id)

# 2
review = Review.create!(content: 'hogehoge', user: user)

# 3
review = user.reviews.create!(content: 'hogehoge')
```

1~3はInsertのSQLは全部同じですが、作成前のSQLと作成後のキャッシュの状態に違いがあります。
irbで確認してみましょう。

```irb:#1
# 作成前のreview数
irb(main):051:0> user.reviews.size
=> 19

# 作成前にuserがselectされている！
irb(main):054:0> review = Review.create!(content: 'hogehoge', user_id: user.id)
   (0.4ms)  BEGIN
  User Load (0.5ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 1 LIMIT 1
  Review Create (0.7ms)  INSERT INTO `reviews` (`content`, `user_id`, `created_at`, `updated_at`) VALUES ('hogehoge', 1, '2020-06-15 14:46:13.461715', '2020-06-15 14:46:13.461715')
   (2.0ms)  COMMIT
=> #<Review:0x000055fc16072188
 id: 20,

# 返却されたreviewにはuserがキャッシュされている
# →作成時に取得していたuserはこのキャッシュをするためか
irb(main):055:0> review.user
=> #<User:0x000055fc16076c60

# 元々のuser.reviewsの方は作成前のキャッシュのままなので数が増えていない
irb(main):056:0> user.reviews.size
=> 19
# 更新するにはreloadが必要
irb(main):057:0> user.reviews.reload.size
  Review Load (0.9ms)  SELECT `reviews`.* FROM `reviews` WHERE `reviews`.`user_id` = 1
=> 20
```

```irb:#2
# 作成前のreview数
irb(main):058:0> user.reviews.size
=> 20

# 作成前のselectなし
irb(main):059:0> review = Review.create!(content: 'hogehoge', user: user)
   (0.4ms)  BEGIN
  Review Create (0.6ms)  INSERT INTO `reviews` (`content`, `user_id`, `created_at`, `updated_at`) VALUES ('hogehoge', 1, '2020-06-15 14:53:06.510290', '2020-06-15 14:53:06.510290')
   (3.4ms)  COMMIT
=> #<Review:0x000055fc16fa6690
 id: 21,

# 返却されたreviewにはuserがキャッシュされている
# →createに渡したuserオブジェクトがキャッシュされているようだ
irb(main):060:0> review.user
=> #<User:0x000055fc16b28b40
 id: 1,

# 元々のuser.reviewsの方は作成前のキャッシュのままなので数が増えていない
irb(main):061:0> user.reviews.size
=> 20
# 更新するにはreloadが必要
irb(main):062:0> user.reviews.reload.size
  Review Load (0.8ms)  SELECT `reviews`.* FROM `reviews` WHERE `reviews`.`user_id` = 1
=> 21
```

```irb:#3
# 作成前のreview数
irb(main):063:0> user.reviews.size
=> 21

# 作成前のselectなし
irb(main):064:0> review = user.reviews.create!(content: 'hogehoge')
   (0.6ms)  BEGIN
  Review Create (0.6ms)  INSERT INTO `reviews` (`content`, `user_id`, `created_at`, `updated_at`) VALUES ('hogehoge', 1, '2020-06-15 14:55:45.393655', '2020-06-15 14:55:45.393655')
   (1.8ms)  COMMIT
=> #<Review:0x000055fc15fd6120
 id: 22,

# 返却されたreviewにはuserがキャッシュされている
irb(main):065:0> review.user
=> #<User:0x000055fc16b28b40
 id: 1,

# user.reviewsにも追加されている
irb(main):066:0> user.reviews.size
=> 22
```

全てのパターンで、createで返却されたreviewオブジェクトはuserオブジェクトをキャッシュしていました。
ただ、1の場合は作成前にselect文が発行されてしまっています。
アソシエーションのオブジェクトを持っているならオブジェクトを渡した方が効率が良さそうです。
また、3の場合のみ`user.reviews`にも作成したreviewが追加されています。

関連データを更新する時もアソシエーションを使った方が、元データにも追加されるので効率よく扱うことができます。
もしアソシエーションを使わない場合も、2のようにアソシエーションはオブジェクト渡しした方が無駄にselect文が発行されないので効率が良さそうです。

# 最後に
キャッシュについてある程度知っているつもりでしたが、[作成のとき]に書いた1,２番のcreateの前にselectが発行される挙動は今まで気付いていませんでした・・・
意識して試してみないとまだまだ気づいていないことがたくさんありそうですね。
