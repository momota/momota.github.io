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
├── README.md
├── Rakefile
├── app
│   └── models
│       └── hoges.rb
├── app.rb
├── config
│   └── database.yml
├── db
│   └── migrate
│       └── 20140510_create_hoges.rb
├── log
│   ├── database.log
│   └── trace.log
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
== 20140510 CreateHoges: migrating ============================================
-- create_table(:hoges)
   -> 0.1159s
== 20140510 CreateHoges: migrated (0.1160s) ===================================
```

問題なければテーブルが作成されているはず。

```
mysql> show tables;
+--------------------------+
| Tables_in_dev_********** |
+--------------------------+
| hoges                    |
| schema_migrations        |
+--------------------------+
2 rows in set (0.00 sec)

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


## CRUD 操作について

ActiveRecordを使って CRUD 操作する。(insert/select/update/delete)

ActiveRecord では、`ActiveRecord::Base` を継承したクラスがDBの1テーブルに対応し、そのクラスの属性がテーブルの各カラムに対応する。
このクラスのことを一般的に「モデル」と呼ぶ。
Rails では、Rails アプリを生成した段階で MVC 別にディレクトリが生成されるので、`app/models` 以下にこの `ActiveRecord::Base` 継承クラスを作る。
今回は Rails ではないのでそれに従う必要はない。が、モデルが増えた場合を考慮すると `app/models` 以下に整理できておいたほうがコードの可読性とかメンテナンスはしやすそうなので、`app/models/hoges.rb` ファイルを作ることにする。
Rails って理にかなっているんだな。


```ruby
class Hoges < ActiveRecord::Base
end
```

この `Hoges` は対応するテーブル名にあわせる。これもCoC。
レコードを単数形で扱うため、テーブル名を複数形にすることが多いみたい。


### Create: テーブルへ insert する

CRUD の C。

`app.rb` をつくる。

モデルを `new` して属性値をセットしてあげればOK。

```ruby
require "active_record"
require "yaml"
require "erb"
require "./app/models/hoges"


db_conf = YAML.load( ERB.new( File.read("./config/database.yml") ).result )

# 開発用DB接続パラメータ読み込み, 接続する
ActiveRecord::Base.establish_connection(db_conf["db"]["development"])


test_name = "momota.txt"
test_url  = "http://momota.github.io/"

hoge = Hoges.new { |h|
  h.name = test_name
  h.url  = test_url
}
p hoge
hoge.save
p hoge
```

なお、`save` しないと insert されない。

こんな感じで生成時にハッシュを渡してもOK。

```ruby
hoge = Hoges.new(:name => test_name, :url => test_url)
hoge.save
```

それでは実行してみる。

```sh
$ bundle exec ruby app.rb
#<Hoges id: nil, name: "momota.txt", url: "http://momota.github.io/", created_at: nil, updated_at: nil>
#<Hoges id: 1, name: "momota.txt", url: "http://momota.github.io/", created_at: "2014-05-10 23:50:46", updated_at: "2014-05-10 23:50:46">
```

上記から、saveしないと `id` や `created_at` などの値が空なので insert されていないことが分かる。

実際にテーブルの内容を見てみよう。ちゃんと insert されている。

```
mysql> select * from hoges;
+----+------------+--------------------------+---------------------+---------------------+
| id | name       | url                      | created_at          | updated_at          |
+----+------------+--------------------------+---------------------+---------------------+
|  1 | momota.txt | http://momota.github.io/ | 2014-05-10 23:50:46 | 2014-05-10 23:50:46 |
+----+------------+--------------------------+---------------------+---------------------+
1 row in set (0.00 sec)
```


### Read: レコードを select する

CRUD の R。


主キーで select する場合は、`find` メソッドを使う。

```ruby
hoges = Hoges.find( 1 )
```

これは以下の SQL と同じ。

```sql
SELECT * FROM hoges where hoges.id = 1 LIMIT 1;
```


主キー以外だと、`find_by` メソッドを使う。

該当するレコードがなければ `nil` が返ってくる。

```ruby
hoges = Hoges.find_by name: "momota.txt"
```

これは以下のような書き方もできる。

```ruby
hoges = Hoges.where(name: "momota.txt").take
```


これらは以下の SQL と同じ。

```sql
SELECT * FROM hoges where hoges.name = "momota.txt" LIMIT 1;
```

`find_by` メソッドを使ってレコードが存在しないときにだけ insert するように `app.rb` を書き変えてみよう。


```diff
-hoge = Hoges.new { |h|
-  h.name = test_name
-  h.url  = test_url
-}
-p hoge
-hoge.save
-p hoge
+rec = Hoges.find_by url: test_url
+unless rec
+  hoge = Hoges.new { |h|
+    h.name = test_name
+    h.url  = test_url
+  }
+  hoge.save
+  puts "すでにデータは insert 済みなのでここには入らない"
+end

+p rec
```

実行すると最終行の `p rec` が呼ばれていることが分かる。

```sh
$ bundle exec ruby app.rb
<Hoges id: 1, name: "momota.txt", url: "http://momota.github.io/", created_at: "2014-05-10 23:50:46", updated_at: "2014-05-10 23:50:46">
```

`find_or_create_by` メソッドによりさらにスマートな書き方ができる。

```diff
-rec = Hoges.find_by url: test_url
-unless rec
-  hoge = Hoges.new { |h|
-    h.name = test_name
-    h.url  = test_url
-  }
-  hoge.save
+rec = Hoges.find_or_create_by( url: test_url ) do |h|
+  h.name = test_name
  puts "すでにデータは insert 済みなのでここには入らない"
end
```

その他いろいろと以下のページが参考になる。

- [Active Record Query Interface](http://edgeguides.rubyonrails.org/active_record_querying.html)



### Update: レコードを update する

CRUD の U。


これはオブジェクトの属性を更新して `save` するだけ。

```ruby
changed_name = "momota.log"
hoges = Hoges.find_by url: test_url
hoges.name = changed_name
hoges.save
```



### Delete: レコードを delete する

CRUD の D。


これも簡単でモデルオブジェクトから `destroy` or `delete` メソッドを呼ぶだけ。`save` は不要。

`delete` はレコードの削除のみなので高速。`destroy` はレコードとオブジェクトも削除してくれるが、`delete` に比べて低速。


```ruby
hoges = Hoges.find( 1 )
hoges.destroy
```

`where` で複数のレコードをひっかけて全部削除したい場合は、`destroy_all` or `delete_all` メソッドを呼ぶ。

```ruby
hoges = Hoges.destroy_all(url: test_url)
```

`find_by` してからでもOK。

```ruby
hoges = Hoges.find_by url: test_url
hoges.destroy_all
```

`app.rb` を書き換えてみる。

```ruby
-rec = Hoges.find_or_create_by( url: test_url ) do |h|
+hoges = Hoges.find_or_create_by( url: test_url ) do |h|
   h.name = test_name
-  puts "すでにデータは insert 済みなのでここには入らない"
 end

-p rec
+puts "[before delete]record count: #{Hoges.count}"
+Hoges.delete_all(url: test_url)
```

実行すると、確かにレコードが削除されている。

```sh
$ bundle exec ruby app.rb
[before delete]record count: 1
[after delete] record count: 0
```

sqlでも確認できる。

```
mysql> select * from hoges;
Empty set (0.00 sec)

```
