---
layout: post
title: "複数の環境で octopress を使ってブログを書く"
date: 2016-01-29 19:03
comments: true
categories: octopress
---

最近、自宅の Mac Mini が重くなってきたので、別環境でブログを書くべく環境を整備した。
5年目選手くらいなので、そろそろ買い換えたいな…


結論としては、以下を参考にやればOK。deploy のとこでちょいハマりしたのでメモ。

- [Clone your Octopess to blog From Two Places](http://blog.zerosharp.com/clone-your-octopress-to-blog-from-two-places/)


<!-- more -->

1. 新しいマシンで git cloneする
-------------------------------

octopress のレポジトリは、`source` と `master` の 2 つのブランチがある。

`source` はその名の通り、編集する markdown ファイルや scss ファイルなどを管
理するブランチ。

`master` は、公開用のHTMLファイルなどを管理するブランチ。`rake generate` で
生成されるファイル群を管理するブランチ。

まずは、任意の場所で `source` ブランチをクローンする。

```sh
$ git clone -b source https://github.com/YOU/YOU.github.io.git blog
```

次に、`master` ブランチを `_deploy` というディレクトリ名でクローンする。

```sh
$ cd blog
$ git clone https://github.com/YOU/YOU.github.io.git _deploy
```

2. インストールする
-------------------

octopress をビルドするため、bundler をインストールする。

```sh
$ gem install bundler
```

別の環境とシェアするので、 rbenv で ruby バージョンを合わせておく。
シェアしなければ octopress が動くバージョンで `.ruby-version` を上書けばよいと思う。

```sh
$ rbenv install X.X.X-pXXX
$ rbenv local X.X.X-pXXX
$ rbenv rehash
```

ビルドする。

```sh
$ bundle install
$ rake setup_github_pages
Enter the read/write url for your repository
(For example, 'git@github.com:your_username/your_username.github.com):
```

自分の github pages のやつを入力すればOK。


3. お試しポストしてみる
-----------------------

```sh
$ rake new_post\['test'\]
#=> source/_post/YYYY-MM-DD-test.markdown が生成される

$ source/_post/YYYY-MM-DD-test.markdown
#=> 記事を編集する

$ rake generate
$ rake preview
#=> http://localhost:4000/にアクセスして記事が生成できていることを確認する

$ rake deploy
```

自分の場合、`rake deploy` でコケた。
`git push` するところでコケたようだ。

```
 ! [rejected]        master -> master (non-fast-forward)
 error: failed to push some refs to '*******************'
 hint: Updates were rejected because the tip of your current branch is behind
 hint: its remote counterpart. Merge the remote changes (e.g. 'git pull')
 hint: before pushing again.
 hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

認証プロキシを通してる環境だったので、そのせいかなーと疑ったけどちがった。
単純に `master` ブランチが conflict してるだけだった。

`git pull` して解決した。

```sh
$ cd _deploy
$ git pull origin master
#=> master ブランチを pull

$ cd ../
$ rake deploy
```

あとは、`source` のほうもcommitしておく。

```
$ git add .
$ git commit -m "posted"
$ git push origin sorce
#=> source ブランチも更新する
```

