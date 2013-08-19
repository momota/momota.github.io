---
layout: post
title: "first_post"
date: 2013-08-19 23:40
comments: true
categories: 
---
# this is my first post using octopress
* create github repository (yourname.github.io)
* install ruby 1.9.3

```sh
    $ rbenv install 1.9.3-pxx
    $ rbenv local 1.9.3-pxx
    $ rbenv rehash
    $ ruby -v	# => 1.9.3-pxx
```

* git clone octopress
    * http://octopress.org/docs/setup/


* configure octopress
    * http://octopress.org/docs/configuring/


* はてなブックマークボタンを設置する
    * `\{\}`はバックスラッシュなしでOK。ここでは本URLに変換されるためバックスラッシュでエスケープしている。

```html
    <div style="float:left">
    <a href="http://b.hatena.ne.jp/entry/\{\{ site.url \}\}\{\{ page.url \}\}" class="hatena-bookmark-button" data-hatena-bookmark-layout="standard" title="このエントリーをはてなブックマークに追加"><img src="http://b.st-hatena.com/images/entry-button/button-only.gif" alt="このエントリーをはてなブックマークに追加" width="20" height="20" style="border: none;" /></a>
    <script type="text/javascript" src="http://b.st-hatena.com/js/bookmark_button.js" charset="utf-8" async="async"></script>
    </div>
```


* deploy github pages
    * http://octopress.org/docs/deploying/github/

