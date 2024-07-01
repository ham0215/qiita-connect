---
title: 1文字aliasをa-z全部登録してみた
tags:
  - Zsh
  - alias
private: false
updated_at: '2022-04-19T12:38:23+09:00'
id: fd7ddc9336f9e6acd4df
organization_url_name: null
slide: false
ignorePublish: false
---
コマンドラインを使っているとよく使うコマンドをaliasに登録することがあると思います。
今回は厳選した26コマンドをa-zまで１文字aliasにを定義してみました。
ちなみにPCはMac, シェルはzshを使っています。

# よく使うコマンドを確認する
せっかく１文字のaliasにするならよく使っているコマンドがいいですよね。
ということで、まずは`history`を使ってよく使っているコマンドを見てみます。

```zsh
# コマンドの最初の1単語で集計
history 1 | awk '{print $2}' | sort | uniq -c | sort -nr

# コマンドの最初の２単語で集計
history 1 | awk '{print $2,$3}' | sort | uniq -c | sort -nr
```

## １単語で集計した結果
左の数字は出現数です。環境固有のコマンドは省いて、いくつか抜粋してみました。
git系コマンドなどすでにaliasを作っていたものは元のコマンドを＃コメントで追記しています。
vim使いなのでviが一位でした！あとgitコマンドとdockerコマンドが多いですね。

```console
4535 vi
3167 gs # git status
2331 dce # docker-compose exec
2182 git
1702 gg # git grep
1532 gd # git diff
1243 gb # git branch
 913 cd
 858 ls
 691 gp # git push
 505 docker-compose
 492 dcb # docker-compose up --build -d
 433 dls # docker containar ls
 241 gc # git checkout
 229 rm
 203 tail
 202 docker
 190 cp
 169 yarn
 143 ch # chrome履歴のインクリメンタルサーチ(https://qiita.com/ham0215/items/cf9a8a0a8aec33158925)
 117 pwd
 105 mv
  92 mysql
  77 mkdir
  75 ssh
  60 history
  55 brew
  51 open
  49 cat
```

## 2単語で集計した結果
1単語のものは除いて抜粋してみました。
やはりgitやdockerコマンドが多いですね。
vi系は２単語目がファイル名でばらけるので1位からは落ちましたが、主にRailsで開発しているのでGemfileなどRails特有のファイル編集はランクインしています。

```console
761 git commit
397 git pull
322 git checkout
285 vi Gemfile
261 git add
224 ls -l
219 vi docker-compose.yml
197 tail -f
175 cp -p
170 git branch
155 vi Dockerfile
117 docker-compose run
108 docker-compose down
 68 vi README.md
 63 rm -rf
 54 vi package.json
 53 docker image
 45 yarn start
 45 git fetch
 41 docker container
 40 docker-compose restart
 36 open .
```

# alias登録
上記のhistory結果を参考にa-zのaliasを登録してみました。
割り当てる1文字のアルファベットから連想できるようにコマンドを紐付けるのは無理があるので、キーボードの位置で似た機能をまとめる方針にしてみました。

また、1文字だと誤作動してしまうこともあるので`rm -rf`のような間違って実行したら危険なコマンドは避けました。

## キーボードレイアウト
キーボードのレイアウトに当てはめると下記の通りです。
<img width="1248" alt="スクリーンショット 2020-02-05 20.32.04.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/83424/64150992-ed5d-3b28-fc76-0762538e33ad.png">
いつでも参照できるようにgithub.ioにページを作りました。
https://ham0215.github.io/alias.html

左２列は割り当て文字との関連は無視してdocker関連のコマンドを固めました。
残りは可能な限り1文字から連想できるように配置しつつ、git関連のコマンドは割り当てが難しかったので右手に固めました。

各コマンドの説明は次の章に記載します。

## 各コマンドの説明
### a
```console
docker-compose up --build -d
```
docker-composeをビルドしてバックグラウンドで起動するコマンド
PC起動後、開発を始めるときによく使います

### b
```console
git checkout -b
```
新しいブランチを作るときに使うコマンド
(switchを使えという声が聞こえてきそうですが無視しよう)

### c
```console
cp -p
```
ファイルコピー
パーミッションなどを保持する`-p`は癖でいつもつけています。

### d
```console
git diff
```
コミットの前にいつも実行する使用頻度が高いコマンドの一つ

### e
```console
ch
```
下記のqiitaに投稿したchromeの履歴をインクリメンタルサーチして開くコマンド
https://qiita.com/ham0215/items/cf9a8a0a8aec33158925

### f
```console
alias f=find . -name
```
カレントディレクトリ配下のファイルを検索します

```console
$ f application*
./app/mailers/application_mailer.rb
./app/models/application_record.rb
./app/javascript/packs/application.js
./app/javascript/stylesheets/application.css
./app/jobs/application_job.rb
./app/controllers/application_controller.rb
./app/views/layouts/application.html.erb
./app/helpers/application_helper.rb
./app/channels/application_cable
./config/application.rb
./config/initializers/application_controller_renderer.rb
```

### g
```console
git grep
```
これも使用頻度が高いコマンドの一つ

### h
下記のファンクションを割り当てました。

```zsh
function gc {
  local c="$(git branch | peco | sed -e 's/* //g' | awk '{print $1}')"
  if [ -n "$c" ]; then
    git checkout $c
  fi
}
```
ローカルブランチをインクリメンタルサーチして、選択したブランチにスイッチするコマンドです
(switchを使えという声が...)

### i
```console
git branch -d
```
ブランチを削除するときに使います

### j
```console
git status
```
git関連コマンドで一番実行するコマンド
無意識にstatus確認してる

### k
```console
git pull
```
git関連はよく使うコマンドが多いですね
featureをマージした後にマージ先ブランチにスイッチして実行するケースが多い

### l
```console
ls -ltr
```
昔からの癖でlsは-ltrを付けて実行することが多い
タイムスタンプが新しいものがリストの下に来るのでコマンドラインでは見やすいです

### m
```console
git commit
```
もはや説明不要

### n
```console
git add
```
もはや説明不要

### o
```console
open .
```
カレントディレクトリをFinderで開きたいときに使っています

### p
下記のファンクションを割り当てました。

```zsh
function gp {
  local c="$(git branch | grep '*' | awk '{print $2}')"
  if [ -n "$c" ]; then
    git push origin $c
  fi
}
```
カレントブランチと同名のリモートブランチにpushするコマンド

### q
```console
docker-compose logs -f
```
各コンテナのログを垂れ流すコマンド

### r
```console
r
```
aliasを設定していません
1文字aliasを設定しようと思ったときに初めて知ったコマンドだが、便利そうだったので残しています
rは直前のコマンドを実行するコマンドです
便利だと思った点は、ただ直前のコマンドを実行するだけではなく、引数に渡したコマンドの中で直前に実行したコマンドを実行してくれるところです
言葉だと説明しづらいですが下記のように動きます

```console
$ history
vi Gemfile
vi config/routes.rb
ls

$ r
=> lsが実行される
$ r vi
=> vi config/routes.rbが実行される 
```

### s
```console
docker-compose restart
```
コンテナの再起動
たまに使います

### t
```console
tail -f
```
ログ垂れ流すときに使うよね

### u
```console
git branch
```
カレントブランチの確認も無駄によくします

### v
下記のファンクションを割り当てました。

```zsh
function gvi {
  vi $(git grep -n $@ | peco --query "$LBUFFER" | awk -F : '{print "-c " $2 " " $1}')
}
```
`v keyword`のようにキーワードを指定して使います
keywordでgit grepした結果をインクリメンタルサーチして、選択したファイルをvimで開きます

### w
```console
docker container ls
```
起動中のコンテナ一覧
コンテナ起動はよく失敗するので起動確認でよく使います
ちなみにwはログインしているユーザーと実行中のプロセスを表示するというコマンドが割り当てられているが、ローカルマシンで使うことはまずないので上書きすることにしました

### x
```console
docker-compose exec
```
コンテナ内でコマンド実行
dockerで開発していると何を実行するにも必要になるコマンド

### y
```console
open https://ham0215.github.io/alias.html
```
キーボードレイアウトに載せた1文字aliasの割り当てを書いたページを開くコマンド
割り当てを忘れたときのヘルプ

### z
```console
docker-compose down
```
コンテナ、使い終わったら終了させましょう（ローカルだと放置しがち）

# 最後に
最後まで読んでいただきありがとうございます。
早速自分の環境に適用していますが1文字aliasに手が慣れるまでしばらくかかりそうです。。。が、慣れればタイプ量がかなり減るはずなのでしばらくはこれでやってみようと思います。
今後、使うコマンドが変わってくると思うので定期的に見直すのが良さそうですね。
