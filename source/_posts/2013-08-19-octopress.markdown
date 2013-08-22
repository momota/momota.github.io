---
layout: post
title: "octopressでblog"
date: 2013-08-19 23:40
comments: true
categories: octopress
---
# 流れ
1. github pages用リポジトリ作成
2. octopressコードをローカルにclone
3. 関連gemをインストール
4. 新規記事をpostする
5. 記事とソースをgithubへデプロイする


# 環境
* mac os x 10.8.4
* git version 1.7.9.6 (Apple Git-31.1)
* zsh


<!-- more -->

# create new github repository
githubへログイン後、github pages用リポジトリを作成する。https://github.com/new
リポジトリ名は、「username.github.io」。


# 最新版のoctopressコードをclone

```sh
    $ git clone git://github.com/imathis/octopress.git username.github.io
    $ cd /path/to/username.github.io
```


# install ruby 1.9.3 by rbenv

```sh
    $ rbenv install 1.9.3-pxx
    $ rbenv local 1.9.3-pxx
    $ rbenv rehash
    $ ruby -v	# => 1.9.3-pxx
```



# configure octopress
bundlerをインストールし、関連gemをインストールする。

* http://octopress.org/docs/setup/
* http://octopress.org/docs/configuring/


```sh
    $ gem install bundler
    $ bundle install --path vendor/bundle
```

_config.ymlを編集する。とりあえず以下のフィールドを自分に合わせて修正。

```yaml
    url: http://momota.github.io
    title: momota.txt
    subtitle: momota memo
    author: momota
    date_format: "%Y-%m-%d"

    email: makoto.momota@gmail.com

    github_user: momota
    twitter_user: momota
    disqus_short_name: momota
    facebook_like: true
```


## install octopress theme

install default theme

```sh
    $ bundle exec rake install
```

install 3rd party theme

https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes

```sh
    $ bundle exec rake install\['theme you like'\]
```


## はてなブックマークボタンを設置する
* source/_includes/post/sharing.html に以下を追加
* `\{\{ site.url \}\}\{\{ page.url \}\}`部分の`\{\}`はバックスラッシュなしでOK。ここでは本URLに変換されるためバックスラッシュでエスケープしている。

```html
    <div style="float:left">
    <a href="http://b.hatena.ne.jp/entry/\{\{ site.url \}\}\{\{ page.url \}\}" class="hatena-bookmark-button" data-hatena-bookmark-layout="standard" title="このエントリーをはてなブックマークに追加"><img src="http://b.st-hatena.com/images/entry-button/button-only.gif" alt="このエントリーをはてなブックマークに追加" width="20" height="20" style="border: none;" /></a>
    <script type="text/javascript" src="http://b.st-hatena.com/js/bookmark_button.js" charset="utf-8" async="async"></script>
    </div>
```


## コメント欄を設ける by disqus
ブログパーツとしてコメント欄を提供する [disqus](http://disqus.com/) のアカウントを作る。
`_config.yml` で設定する。

```yml
    # Disqus Comments
    disqus_short_name: xxxxxx
    disqus_show_comment_count: false
```



# post

* 以下コマンドで、記事ファイル`source/_posts/yyyy-MM-dd-post-title.markdown`を生成。
* 生成後は、vim/emacsなど自分の好きなエディタで編集する

```sh
    $ bundle exec rake new_post\['post title'\]
```

# check your post

```sh
    # htmlなどの生成
    $ bundle exec rake generate
    # プレビュー: http://localhost:4000/ へアクセスしてみる
    $ bundle exec rake preview
```


# deploy github pages
* http://octopress.org/docs/deploying/github/ を参考に。

```sh
    $ bundle exec rake setup_github_pages
    # Enter the read/write url for your repository:
    # --> git@github.com:username/username.github.io.git
    $ bundle exec rake deploy
```

* ソースもgit pushする

```sh
    $ git add .
    $ git commit -m 'commit comment'
    $ git push origin source
```

