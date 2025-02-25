---
title: "Rails(Natto) × MeCab × Unidic の環境を Docker で構築する"
emoji: "🏞️"
type: "tech"
topics:
  - "rails"
  - "ruby"
  - "形態素解析"
  - "mecab"
published: true
published_at: "2024-03-19 13:22"
---

Rails で構築しているアプリケーションで分かち書きを行いたかったので、形態素解析エンジンである MeCab の検証を行いました。MeCab を選択した理由としては、Ruby に限らず広く使われていることと、バインディングである Natto が存在しているためです。

今回は、Docker で Rails と MeCab の環境構築を行い、Rails console で Natto 経由で MeCab による分かち書きをおこなうところまでを記事にしました。
MeCab のインストールに加え、`docker compose build` したときに、システム辞書として、UniDic が適用されるような設定も追加しています。

前半は Rails の環境構築についてのメモになっています。「MeCab の導入について」からMeCab の導入に関する解説になっています。

# MeCab とは
京都大学情報研究学科と NTT コミュニケーション科学基礎研究所の共同研究を通じて開発されたオープンソースの形態素解析エンジンで、辞書やコーパス(文章を構造化し、大規模に集積したもの)に依存しない汎用的な設計と高速な動作と解析精度の高さなどが特徴です。名前は作者である工藤拓さんの好物であることが由来だそうです。

参考 https://taku910.github.io/mecab/

# サンプルリポジトリ

https://github.com/ndjndj/rails-mecab

# ディレクトリ構造
こんな感じでプロジェクトディレクトリに空のファイル群を作成しておきます。

```
.
├─ compose.yml
└─ rails
    ├─ Dockerfile
    ├─ Gemfile
    ├─ Gemfile.lock
    └─ entrypoint.sh
```

データベースに `postgres` を使用していますが、今回は使用しないので任意で大丈夫です。
`Gemfile.lock` は空のままで大丈夫です。
`entrypoint.sh` には、サーバーのプロセスが残っている場合、新たにサーバーを起動できないため、それを防ぐために起動時にプロセスをいったん削除するコマンドが記述されています。

:::details compose.yml
```yml
version: '3'
services: 
  db: 
    container_name: postgres-sample
    image: postgres:16.0
    ports: 
      - "5432:5432"
    volumes: 
      - pg-data:/var/lib/postgresql/data
    environment: 
      - POSTGRES_DATABASE=postgres 
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password 
      - POSTGRES_ROOT_PASSWORD=root
  api: 
    container_name: api-sample
    build: 
      context: ./rails
    command: bash -c "rails s -b '0.0.0.0'"
    volumes:
      - ./rails:/usr/src/app
    ports: 
      - 3000:3000
    depends_on:
      - db 
    tty: true 
    stdin_open: true
volumes: 
  pg-data:
  bin: 
    driver: local
```
:::

:::details Dockerfile
```Dockerfile
FROM ruby:3.3.0
RUN apt-get update -qq && apt-get install -y vim 

RUN apt-get install libmecab2 libmecab-dev mecab mecab-ipadic mecab-ipadic-utf8 mecab-utils
RUN apt-get install -y unidic-mecab
RUN apt-get install -y build-essential

RUN mkdir /usr/src/app 
WORKDIR /usr/src/app
COPY Gemfile /usr/src/app/Gemfile 
COPY Gemfile.lock /usr/src/app/Gemfile.lock 
RUN sed -i 's/^dicdir/;dicdir/' /etc/mecabrc
RUN echo "\ndicdir = /var/lib/mecab/dic/unidic" >> /etc/mecabrc

RUN gem update --system 
RUN bundle update --bundler 

RUN bundle install 
COPY . /usr/src/app 

COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh 
ENTRYPOINT ["entrypoint.sh"]
```
:::

:::details Gemfile
```
source "https://rubygems.org"
git_source(:github) {|repo| "https://github.com/#{repo}.git" }

ruby "3.3.0"

gem "rails", "~> 7.1.2"
```
:::

:::details entrypoint.sh
```bash
#!/bin/bash
set -e

rm -f /usr/src/app/tmp/pids/server.pid 

exec "$@"
```
:::

# Rails アプリケーションの作成

下記コマンドを実行して、`Rails` アプリケーションを作成します。

```bash 
docker compose run --rm api rails new . --skip --database=postgresql --api --skip-bundle
```

:::message
.git フォルダが作成されるので消去します。
:::

# Gemfile, database.yml を書き換える
以下のように `Gemfile` を書き換えます。
また、`config/database.yml` も作成した環境に合わせて書き換えます。

:::details Gemfile
```Gemfile
source "https://rubygems.org"
git_source(:github) {|repo| "https://github.com/#{repo}.git" }

ruby "3.3.0"

gem "bootsnap", require: false

gem "mecab"

gem "natto"

gem "pg", "~> 1.1"

gem "puma", "~> 6.4.0"

gem "rails", "~> 7.1.2"

gem "tzinfo-data", platforms: %i[mingw mswin x64_mingw jruby]

```
:::

:::details config/database.yml
```yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: postgres
  password: password 
  host: db
  port: 5432

development:
  <<: *default
  database: postgres_dev
  password: password

test:
  <<: *default
  database: postgres_test
  password: password
```
:::

# gem をインストールする
下記コマンドを実行して、gem をインストールします。

```bash
docker compose run --rm api bundle install
```

# Docker コンテナを立ち上げる
コンテナをビルドして立ち上げます。

```bash
docker compose build --no-cache
docker compose up -d
```

# Postgresql の DB を作成する
現状だと DB が存在しないため作成する必要があります。
`rails` コンテナ環境に入ります。

```bash
docker compose exec api /bin/bash
```

`rails` コンテナ内で `db:create` して DB を作成します。

```
rails db:create
```

# MeCab の導入について
Dockerfile に記述を追加しています。
冒頭に掲載していますが、該当する部分を再度下記に記します。

```
# Dockerfile
# (中略)
RUN apt-get install libmecab2 libmecab-dev mecab mecab-ipadic mecab-ipadic-utf8 mecab-utils
RUN apt-get install -y unidic-mecab
RUN apt-get install -y build-essential

# (中略)
RUN sed -i 's/^dicdir/;dicdir/' /etc/mecabrc
RUN echo "\ndicdir = /var/lib/mecab/dic/unidic" >> /etc/mecabrc

# (中略)
```

## MeCab 基本ライブラリの導入

```
RUN apt-get install libmecab2 libmecab-dev mecab mecab-ipadic mecab-ipadic-utf8 mecab-utils
```

この辺の記事を参考にさせていただきました
参考 https://www.kkaneko.jp/tools/ubuntu/ubuntu_mecab.html
参考 https://qiita.com/Takayoshi_Makabe/items/18cefa4b4572d12b5aa9
参考 https://qiita.com/SUZUKI_Masaya/items/685000d569452585210c
参考 https://riocampos-tech.hatenablog.com/entry/20150413/use_mecab_with_ruby_on_mac

## UniDic の導入
UniDic は日本語テキストを単語に分割し、形態論情報を付与するための電子化辞書です。国立国語研究所がメンテナンスしており、MeCab でも利用可能な解析用の UniDic を公開・配布してくれています。
ちなみに、MeCab 用のシステム辞書は他にも `NEologd` や、`ipadic` などがありますが、どれも少なくともここ3-4年は更新が停止されており、現在もメンテナンスが続けられているのは、自分が調べた限りでは UniDic のみでした。

```
RUN apt-get install -y unidic-mecab
```

参考 https://clrd.ninjal.ac.jp/unidic/
参考 https://csd.ninjal.ac.jp/lrc/?UniDic

## mecabrc の設定変更
ここが今回一番はまった個所でした。
mecabrc では、MeCab で利用するシステム辞書や他設定を規定することができます。
下記にデフォルトの mecabrc を置いておきます。

```
;
; Configuration file of MeCab
;
; $Id: mecabrc.in,v 1.3 2006/05/29 15:36:08 taku-ku Exp $;
;
dicdir = /var/lib/mecab/dic/debian

; userdic = /home/foo/bar/user.dic

; output-format-type = wakati
; input-buffer-size = 8192

; node-format = %m\n
; bos-format = %S\n
; eos-format = EOS
```

デフォルト辞書は `dicdir` で規定されており、ここを先ほどインストールした UniDic に変更する必要があります。
新たにインストールした辞書は、デフォルト辞書と同じ階層にあるはずです。今回なら、`/var/lib/mecab/dic/unidic` を指定します。
つまり、既存の設定を無効にして、新しく `dicdir = /var/lib/mecab/dic/unidic` を追記します。これが、Dockerfile の下記部分です。

```
RUN sed -i 's/^dicdir/;dicdir/' /etc/mecabrc
RUN echo "\ndicdir = /var/lib/mecab/dic/unidic" >> /etc/mecabrc
```

当初、設定を書き換えた `mecabrc` ファイルをローカルに作成しておいて `COPY` コマンドで mecabrc ファイルを置換するように Dockerfile に記述していたのですが、`[pos != std::string::npos] format error:` が発生して MeCab を起動できなくなってしまったので、ファイルの置換ではなくファイルの中身の書き換えを行うようにしました。

# Rails console で確認
Rails コンソールを利用して Natto 経由で MeCab を実行して、デフォルト辞書が UniDic になっていることと分かち書きができることを確認します。

```rb
require "natto"
nm = Natto::MeCab.new
text = "改善の余地はあるさ。"
puts nm.parse(text)
nm.dicts
# => [#<Natto::DictionaryInfo:0x000 @filepath="/var/lib/mecab/dic/unidic/sys.dic", charset=UTF-8, type=0>]
```

参考 https://github.com/buruzaemon/natto
参考 https://rubydoc.info/gems/natto/Natto/MeCab
参考 https://qiita.com/k-shogo/items/0f8a98c52913c729c7eb

# 参考
https://taku910.github.io/mecab/
https://www.kkaneko.jp/tools/ubuntu/ubuntu_mecab.html
https://qiita.com/Takayoshi_Makabe/items/18cefa4b4572d12b5aa9
https://clrd.ninjal.ac.jp/unidic/
https://github.com/buruzaemon/natto
https://rubydoc.info/gems/natto/Natto/MeCab
https://qiita.com/k-shogo/items/0f8a98c52913c729c7eb

# 他学習に参考になった記事
https://www.jnlp.org/nlp/%E5%BD%A2%E6%85%8B%E7%B4%A0%E8%A7%A3%E6%9E%90
http://vdeep.net/mecab-natto
https://qiita.com/Haruka-Ogawa/items/f261b8b03320f2fba9f2
https://zenn.dev/t_o_d/articles/a7ae37c353c072ff2874