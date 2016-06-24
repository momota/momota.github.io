---
layout: post
title: "ネットワークエンジニアのための cisco.vim"
date: 2016-06-24 16:42
comments: true
categories: cisco catalyst ios vim network
---

ネットワークエンジニアのみなさん、こんにちは。

[ネットワークエンジニアのための junos.vim](http://momota.github.io/blog/2016/06/22/junos-dot-vim/) の cisco バージョンです。


cisco の configuration をカラーリングする vim プラグイン
[momota/cisco.vim](https://github.com/momota/cisco.vim) を作ったのでご活用下さい。

対象は、cisco ルータと Catalyst スイッチ、Nexus スイッチです。


色を付けることによって、configuration の解釈を手助けしたり、
configuration 作成時のミスに気づきやすくしたりする効果が期待できます。
(誤ったタームを入力すると色が付かないので)


以下のようにカラーリングします。 (colorscheme は molokai)

before
------

![display_before](/images/20160624_cisco-config_before.png)

after
-----

![display_after](/images/20160624_cisco-config_after.png)


<!-- more -->

インストール
------------

NeoBundle で vim プラグインを管理している人は以下の行を`.vimrc` に追加して

```vim
NeoBundle 'momota/cisco.vim'
```

`:NeoBundleInstall` でインストールする。

その他は、[README/Installation](https://github.com/momota/cisco.vim#installation) を参照。


使い方
------

cisco config ファイルを `.cisco` という拡張子で保存して vim で開くか、
cisco config ファイルの先頭に以下行を追加して保存し vim で開くと色付けしてくれる。

```
! vim: set ft=cisco:
```

もしくは、cisco config ファイルを vim 開いている時に `:set ft=cisco` を実行する。


関連
====

- [ネットワークエンジニアのための junos.vim](http://momota.github.io/blog/2016/06/22/junos-dot-vim/)

