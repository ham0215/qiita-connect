---
title: 汎用的な機能を実装するときは使う側が簡単に使えるか？を意識しよう
tags:
  - Ruby
  - オブジェクト指向
  - クラス
  - モジュール
  - 共通化
private: false
updated_at: '2022-04-19T12:33:17+09:00'
id: b3938c5ee6fab08fceaa
organization_url_name: null
slide: false
ignorePublish: false
---
プログラムを書いているとき、汎用的な機能をクラスやメソッドに切り出して実装することがあると思います。

例えば指定した複数のユーザーを特定のイベントに参加させる処理を考えます。

まずはすべての処理が1メソッドにある場合の実装を確認します。
（Rubyで書いていますが他のプログラム言語をやったことがある方であれば雰囲気はつかめると思います）

```ruby
def main(user_ids, event_id)
  users = User.where(id: user_ids)
  event = Event.find(event_id)

  # 参加しているユーザーは除く
  target_users = users.reject {|user| user.events.ids.include?(event.id) }

  # イベント参加
  target_users.each do |user|
    # イベント参加 => EventUserにレコードを追加
    EventUser.create!(user: user, event: event)
  end
end
```

上記からイベント参加の処理をメソッドとして切り出してみます。
今回はEventモデルにメソッド追加してみました。

```ruby
def main(user_ids, event_id)
  users = User.where(id: user_ids)
  event = Event.find(event_id)

  # 参加しているユーザーは除く
  target_users = users.reject {|user| user.events.ids.include?(event.id) }

  # イベント参加
  event.join(target_users)
end
```

```ruby:app/models/event.rb
def join(users)
  users.each do |user|
    # イベント参加 => EventUserにレコードを追加
    EventUser.create!(user: user, event: event)
  end
end
```

これで他のところからイベントに参加する処理を実行したくなった場合も`event.join(users)`で呼び出すことができるようになりますね。

ただ、この実装方法は微妙な点があります。
それは呼び出し元が**参加済みのユーザーを除いて参加可能なユーザーのみを渡さないといけない**点です。

今回の例の場合は下記のようにしたほうが呼び出し元が楽になります。

```ruby
def main(user_ids, event_id)
  users = User.where(id: user_ids)
  event = Event.find(event_id)

  # イベント参加
  # 呼び出し元はusersが参加済みか気にしなくて良い。対象となりうるユーザーを渡せばOK
  event.join(users)
end
```

```ruby:app/models/event.rb
def join(users)
  # 参加しているユーザーは除く
  target_users = users.reject {|user| user.events.ids.include?(event.id) }

  target_users.each do |user|
    # イベント参加 => EventUserにレコードを追加
    EventUser.create!(user: user, event: event)
  end
end
```

このように実装すれば呼び出し元は参加済みか気にせずに対象となりうるユーザーを渡せば良くなります。

今回のように順序立って説明すると3つ目の実装をするのが当たり前のように感じるかもしれませんが、実際には2つ目の状態で実装されているものもよく見かけます。

なぜならメソッド単体で見て「ユーザーをイベント参加させるメソッドです」と言われても違和感がないからです。
もう一度2つ目の汎用化した部分を下記に再掲します。このメソッド単体で見るとどうでしょうか？

```ruby:app/models/event.rb
def join(users)
  users.each do |user|
    # イベント参加 => EventUserにレコードを追加
    EventUser.create!(user: user, event: event)
  end
end
```

また、最初に汎用メソッドを作ったときは呼び出し元がほとんどない（もしくは1つしかない）かもしれません。
そうなると呼び出し元でチェックしても呼び出し先でチェックしても手間はあまり変わらないので、呼び出し元ですべてでチェックしなければいけないという煩わしさにも感じづらいです。

そして一度このように実装されてしまうと、同じようにイベント参加をする処理を実装するときも既存の箇所をコピペしてしまいます。
そうなると『既存がこうだから〜』の力学が働いてしまい、煩わしさを感じるどころか当たり前と感じるようになってきます。

# まとめ
汎用的な機能を実装するときは、使う側が簡単に使えるか？を意識するようにしましょう。
使う側が様々な制約を考慮しなければいけなかったり、必ず実行しなければいけない処理（今回の例でいうと参加者を除く処理）がある場合、それらの処理を汎用的な機能の内部に取り込めないか検討するようにしましょう。

そうすることで使う側が楽になるだけでなく、実装漏れなどのバグも防ぐことに繋がります。

また、せっかく汎用的に作るのであれば、使われてなんぼです。
使い方が難しいと実装した人以外は使えない可能性が高いです（実装した人も数カ月後には使えなくなっているかもしれません）
利用者が使いやすくて楽ができる実装を心がけましょう！！
