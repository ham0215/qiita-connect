---
title: '[Rails]モデルのバリデーション「lengthチェック」の閾値はテーブル情報から動的に取得しよう'
tags:
  - Ruby
  - Rails
  - ActiveRecord
  - RDB
private: false
updated_at: '2022-04-19T12:32:42+09:00'
id: d061decdb820b28ab6db
organization_url_name: null
slide: false
ignorePublish: false
---
下記のusersテーブルがあるとします。

```console
mysql> desc users;
+-----------------------------+-------------+------+-----+---------+----------------+
| Field                       | Type        | Null | Key | Default | Extra          |
+-----------------------------+-------------+------+-----+---------+----------------+
| id                          | bigint(20)  | NO   | PRI | NULL    | auto_increment |
| name                        | varchar(100)| NO   |     | NULL    |                |
| created_at                  | datetime    | NO   |     | NULL    |                |
| updated_at                  | datetime    | NO   |     | NULL    |                |
+-----------------------------+-------------+------+-----+---------+----------------+
```

この場合、RailsのUserモデルを作るときにnameには100文字未満というバリデーションを入れることが多いと思います。
そのときに下記のように100とハードコーディングしていませんか？

```ruby:app/models/user.rb
validates :name, presence: true, length: { maximum: 100 }
```

Railsではデータベーステーブルのメタ情報を簡単に取得することができます。
これを使うことで下記のように100の部分をテーブル定義から動的に取ることができます。

```ruby
validates :name, presence: true, length: { maximum: columns.find{|c| c.name == 'name' }.limit }
```

このように書いておくことでテーブル定義を変更したときも自動的にバリデーションの値も更新されるのでおすすめです。
