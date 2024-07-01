---
title: 全文検索(N-gram)がRSpecでうまく動かなかった話
tags:
  - Rails
  - MySQL
  - RSpec
  - n-gram
private: false
updated_at: '2022-04-19T12:23:26+09:00'
id: 3e96ec35eeab7bf06cfe
organization_url_name: null
slide: false
ignorePublish: false
---
全文検索(N-gram)を使っているシステムの開発に入るためにローカル環境を構築していたときにrspecがうまく動かずに少しハマったので原因と対策を書きます。

# 事象
githubからソースを落としてきて、諸々設定を行い、データベースも整えて、rspec実行！！

```bash
$ git clone git@repo
~~~ 諸々環境設定 ~~~
$ bundle install
# develop_db, test_dbを作成
$ bundle exec rake db:create
# develop_dbにマイグレーション
$ bundle exec rake db:migrate
# test_dbにdevelop_dbのスキーマを反映
$ bundle exec rake db:test:prepare
# rspec!!
$ bundle exec rspec
.......F...F...
```
なぜか一部通らない。なにか設定忘れあるのかな。

なお開発環境の各種バージョンは下記の通り

|  | version |
|:-----------------|:-----:|
| ruby | 2.4.1 |
| rails | 5.2.3 |
| mysql | 5.7.10 |

# 調査
rspecが通らなかったテストケースを確認したところ、N-gramを使った全文検索のテストが軒並み落ちている様子。

railsコンソールで該当処理をしてみたところうまくいく。

```ruby
> NGram.where('MATCH(text1) AGAINST (? IN BOOLEAN MODE)', "+テスト")
#<NGram id: 1, text1: 'テスト', ...>
```

だがrspecのエラーをみる限り、全文検索でデータが取れていないように見える。
コンソールでできてrspecでできないということはDBの差が怪しいので`RAILS_ENV=test`を指定してコンソールを起動。

```ruby
> NGram.where('MATCH(text1) AGAINST (? IN BOOLEAN MODE)', "+テスト")
nil
```
データはあるのに取得できていない。
test_dbが怪しいのでdevelop_dbとの差を調べることに。

最初は文字コードがおかしいのかと思い調べてみるも、特に差分なし

```sql
mysql> show variables like '%char%'; -- develop_dbもtest_dbも同じ結果
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8mb4                    |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | latin1                     |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)
```

次にテーブル定義を調べたところ、差分あり。
下記の通りtest_dbにはindexのところに` /*!50100 WITH PARSER ngram */`の記述がない。

```sql
mysql> show create table n_grams;
-- 差分のみ表
-- develop_db
FULLTEXT KEY `text1_ft_index` (`text1`) /*!50100 WITH PARSER `ngram` */
-- test_db
FULLTEXT KEY `text1_ft_index` (`text1`)
```

上記差分を修正してテストを実行したところ無事に全て成功しました。

```bash
$ bundle exec rspec
...............
```

# 原因の考察
テーブル定義に差分ができた理由としては下記が考えられる。

* N-gramの記述はRailsで対応できていない。Alterするときも下記のようにSQLをそのまま書く必要がある

```sql
ALTER TABLE n_grams ADD FULLTEXT text1_ft_index (text1) WITH PARSER ngram;
```

* test_dbは`rake db:test:prepare`で作成したのでmigrationファイルではなくschema.rbを基にテーブルを作っている
* 上記の通りN-gramの記述はRailsは対応できていないのでschema.rbにはN-gramが表現できていない
* schema.rbから作られたtest_dbはN-gramの設定がうまく反映されなかった
* develop_dbはマイグレーションファイルを基に作成したのでN-gramの設定が反映された

test_dbもマイグレーションファイルから生成するようにしたら解決しました。

```
$ RAILS_ENV=test bundle exec rake db:migrate
```

## 2019/10/25 追記
もっと良い方法を見つけたので追記。
それはスキーマファイルをSQLで保存する方法です。
下記のようにフォーマットを指定できます。デフォルトは`ruby`です。

```sql:config/application.rb
  config.active_record.schema_format = :sql
```

この指定を入れてからマイグレーションすると`structure.sqlが生成されます。
中身をみてみると、N-gramの記述も反映されています。

```sql:db/structure.sql
FULLTEXT KEY `text1_ft_index` (`text1`) /*!50100 WITH PARSER `ngram` */
```

この状態であれば`rake db:test:prepare`でも正しくテーブルが作成されました。
こちらの方が標準のやり方に則っているので良いと思います。

### structure.sqlを使う注意点
structure.sqlを使う際、２点ほど注意点があります。MySQLの場合を記載していますが他も同様だと思います。

* rake db:migrateでstructure.sqlを出力するためにmysqldumpコマンドがインストールされている必要がある
* rake db:test:prepareでmysqlコマンドがインストールされている必要がある
