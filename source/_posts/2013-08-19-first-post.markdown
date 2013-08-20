---
layout: post
title: "octopress"
date: 2013-08-19 23:40
comments: true
categories: 
---
octopress setting
=================

## create new github repository
githubへログイン後、github pages用リポジトリを作成する。
リポジトリ名は、「username.github.io」。
https://github.com/new を参考に。


## install ruby 1.9.3 by rbenv

```sh
    $ rbenv install 1.9.3-pxx
    $ rbenv local 1.9.3-pxx
    $ rbenv rehash
    $ ruby -v	# => 1.9.3-pxx
```



## configure octopress
* http://octopress.org/docs/setup/
* http://octopress.org/docs/configuring/
* 最新版のoctopressコードをクローンする。

```sh
    $ git clone git://github.com/imathis/octopress.git username.github.io
```

* bundlerをインストールし、関連gemをインストールする。

```sh
    $ cd /path/to/username.github.io
    $ gem install bundler
    $ bundle install --path vendor/bundle
```

### install octopress theme

* default theme

```sh
    $ bundle exec rake install
```

* 3rd party theme
https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes

```sh
    $ bundle exec rake install\['theme you like'\]
```


### はてなブックマークボタンを設置する
* source/_includes/post/sharing.html に以下を追加
* `\{\{ site.url \}\}\{\{ page.url \}\}`部分の`\{\}`はバックスラッシュなしでOK。ここでは本URLに変換されるためバックスラッシュでエスケープしている。

```html
    <div style="float:left">
    <a href="http://b.hatena.ne.jp/entry/\{\{ site.url \}\}\{\{ page.url \}\}" class="hatena-bookmark-button" data-hatena-bookmark-layout="standard" title="このエントリーをはてなブックマークに追加"><img src="http://b.st-hatena.com/images/entry-button/button-only.gif" alt="このエントリーをはてなブックマークに追加" width="20" height="20" style="border: none;" /></a>
    <script type="text/javascript" src="http://b.st-hatena.com/js/bookmark_button.js" charset="utf-8" async="async"></script>
    </div>
```


## deploy github pages
* http://octopress.org/docs/deploying/github/ を参考に。

```sh
    $ bundle exec rake setup_github_pages
    # Enter the read/write url for your repository:
    # --> git@github.com:username/username.github.io.git
    $ bundle exec rake generate
    $ bundle exec rake deploy
```

ソースもgit pushする

```sh
    $ git add .
    $ git commit -m 'commit comment'
    $ git push origin source
```

