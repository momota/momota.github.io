---
layout: post
title: "rails じゃなくても ActiveRecord を使う"
date: 2014-05-10 14:54
comments: true
categories: ruby activerecord rails rake
---
rails には挫折したおれが、rails アプリケーション以外で ActiveRecord を使うようになった件について。

ActiveRecord は O/Rマッパーで RDB のテーブルエントリをオブジェクトとして扱えるようにするやつ。1インスタンスが、テーブルの1レコードに相当する。
Ruby on Rails 標準で、モデル層で使われる。

環境は以下。

- Mac OSX
- ruby 2.1.1p76
- MySQL mysqld5.6.17 


最終的なディレクトリ構成は以下のようになる。

```sh
.
├── Gemfile
├── Gemfile.lock
├── Rakefile
├── config
│   └── database.yml
├── db
│   └── migrate
│       └── 20140510_create_hoges.rb
├── log
│   ├── database.log
│   └── trace.txt
└── vendor
    └── bundle
            └── (略)
```

- https://github.com/momota/activerecord_sample


<!-- more -->

## 前準備


### ruby

rbenv で ruby 2.1.1 をインストールする。

```sh
$ rbenv install 2.1.1
$ rbenv local 2.1.1
$ rbenv rehash
$ ruby -v
ruby 2.1.1p76 (2014-02-24 revision 45161) [x86_64-darwin13.0]
```

### gem

bundler で gem をインストールする。
まず、bundler をインストールする。

```sh
$ gem install bundle
$ bundle -v
Bundler version 1.6.2
```

以下の `Gemfile` をつくって`bundle install --path vendor/bundle` して gem をインストールする。

```ruby
source "https://rubygems.org"

gem "activerecord"
gem "rake"
gem "mysql2"
gem "pry"
```

`mysql2` はDBアダプタ。MySQLと接続するために必要。

`pry` はなくてもよい。`irb` の便利バージョン。


### mysql

homebrew でインストールした。

`my.cnf`はよしなに。


`create database` とか `grant` でユーザやデータベースなどはあらかじめ作っておく。



## create table: rake タスクを使ってテーブルを作成する

`rake` (makeみたいなもん) でテーブルを作成する。(マイグレーション)

マイグレーションは、SQLを使わずにデータベースのテーブルやカラムなどの構造を変更できる仕組みで、移行と解釈するとややこしい。データベース移行をしやすくする仕組み、くらいに捉えておくとよい。


### マイグレーション用ファイルを作成する

`db/migrate` ディレクトリを作ってマイグレーション用ファイルを作る。

ここでは hoges テーブルを作ることにする。
マイグレーション用ファイルには、テーブル定義を書く。ここでは `db/migrate/20140510_create_hoges.rb` を作成する。

このファイル名が大事で `20140510` の部分がバージョンとして管理される。`schema_migrations` テーブルが自動生成され、そこで管理される。また、マイグレーション用ファイル中に class 名を`CreateHoges` のようにキャメルケースで定義した場合は、ファイル名は `VERSION_create_hoge.rb` のようにスネークケースとして命名する必要がある。ファイル名がこの命名規約に反するとマイグレーションがこける。

rails文化の「設定より規約」(CoC: Convention over Configuration)ってやつですね。


`ActiveRecord::Migration` を継承したクラス `Createhoges` を定義する。
シンボル `:hoges` がテーブル名。カラム定義は見ての通りだと思う。
主キーは自動的に `id` というカラム名で生成されるので書かない。(書くとエラーになる)


```ruby
class CreateHoges < ActiveRecord::Migration
  def self.up
    create_table :hoges do |t|
      t.string  :name
      t.string  :url
      t.timestamps  # => これでcreated_atとupdated_atカラムが定義される
    end
  end

  def self.down
    drop_table :hoges
  end
end
```

書き方の詳細は、[Active Record Migrations](http://guides.rubyonrails.org/migrations.html) を見れば良いと思う。


### データベース接続情報を yaml に書き出す

MySQL への接続情報(DBユーザ名とかパスワードとか使うDB名とか)を yaml ファイルに書き出しておく。めんどくさい、かつ、使い捨てのコードならハードコーディングしといてもOK。

ここでは、以下の内容で `config/database.yml` を作成した。

```erb
db:
  production:
    adapter:  mysql2
    host:     localhost
    username: <%= ENV['DATABASE_USERNAME'] %>
    password: <%= ENV['DATABASE_PASSWORD'] %>
    database: <%= ENV['DATABASE_NAME']%>

  development:
    adapter:  mysql2
    host:     localhost
    username: <%= ENV['DEV_DATABASE_USERNAME'] %>
    password: <%= ENV['DEV_DATABASE_PASSWORD'] %>
    database: <%= ENV['DEV_DATABASE_NAME']%>
```

パスワードなどの秘匿情報は環境変数から読み込むようにする。(ここではERB形式で書いた)
パスワードをべた書きしといて、間違えて github とかで公開しちゃうと大変なので。

`~/.zshrc` とか `~/.bashrc` にあらかじめ作っておいたデータベース名とかユーザ名を以下のように足して `source ~/.zshrc` で読み込めばいいと思う。

```sh
export DEV_DATABASE_NAME="hoge_db"
export DEV_DATABASE_USERNAME="hoge_user"
export DEV_DATABASE_PASSWORD="hoge_password"
```


### Rakefile をつくって rakeタスクを実行する

以下の内容で `Rakefile` を作る。
DB接続用の設定や環境指定(development/production) やバージョン指定の設定を書いてます。

```ruby
require "active_record"
require "yaml"
require "erb"
require "logger"


task :default => :migrate

desc "Migrate database"
task :migrate => :environment do
  ActiveRecord::Migrator.migrate('db/migrate', ENV["VERSION"] ? ENV["VERSION"].to_i : nil)
end

task :environment do
  db_conf = YAML.load( ERB.new( File.read("./config/database.yml") ).result )

  # `rake ENV=development`/`rake ENV=production`で切り替え可能
  ActiveRecord::Base.establish_connection( db_conf["db"][ENV["ENV"]] )
  ActiveRecord::Base.logger = Logger.new("log/database.log")
end
```

以下を参考。

- [非Rails AppでActiveRecord::Migrationを使う + Rakeでバージョン管理する](http://qiita.com/foloinfo/items/6ecfe3c5fd1b56f1dceb)


### rake タスクを実行する

まずは rake タスクの確認。

```sh
$ bundle exec rake -T
rake migrate  # Migrate database
```

開発環境設定で実行する。(ENV=development)

```sh
# debug 用に--traceオプションをつけ、標準エラーをlog/trace.txtへリダイレクト。
# bundle exec rake ENV=development でもOK
$ bundle exec rake ENV=development --trace 2> log/trace.txt
```

問題なければテーブルが作成されているはず。

```
mysql> desc schema_migrations;
+---------+--------------+------+-----+---------+-------+
| Field   | Type         | Null | Key | Default | Extra |
+---------+--------------+------+-----+---------+-------+
| version | varchar(255) | NO   | PRI | NULL    |       |
+---------+--------------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql> select * from schema_migrations;
+----------+
| version  |
+----------+
| 20140510 |
+----------+
1 row in set (0.00 sec)

mysql> desc hoges;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | int(11)      | NO   | PRI | NULL    | auto_increment |
| name       | varchar(255) | YES  |     | NULL    |                |
| url        | varchar(255) | YES  |     | NULL    |                |
| created_at | datetime     | YES  |     | NULL    |                |
| updated_at | datetime     | YES  |     | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
5 rows in set (0.01 sec)

mysql> select * from hoges;
Empty set (0.00 sec)
```



## select とか CRUD 操作について

疲れたのであとで書く。

