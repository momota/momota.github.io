---
layout: post
title: "ネットワークエンジニアのための junos.vim"
date: 2016-06-22 10:57
comments: true
categories: juniper junos vim
---


ネットワークエンジニアのみなさん、こんにちは。

Juniper の configuration をカラーリングする vim プラグイン
[momota/junos.vim](https://github.com/momota/junos.vim) を作ったのでご活用下さい。

色を付けることによって、junos configuration の解釈を手助けしたり、
configuration 作成時のミスに気づきやすくしたりする効果が期待できます。
(誤ったタームを入力すると色が付かないので)


以下のようにカラーリングします。 (colorscheme は molokai)

before
------

![display_before](/images/20160622_junos-confg_before.png)

after
-----

![display_after](/images/20160622_junos-confg_after.png)

before: `display set` モード
----------------------------

![display-set_before](/images/20160622_junos-confg-set_before.png)

after: `display set` モード
---------------------------

![display-set_after](/images/20160622_junos-confg-set_after.png)

<!-- more -->

インストール
------------

NeoBundle で vim プラグインを管理している人は以下の行を`.vimrc` に追加して

```vim
NeoBundle 'momota/junos.vim'
```

`:NeoBundleInstall` でインストールする。

その他は、[README/Installation](https://github.com/momota/junos.vim#installation) を参照。


使い方
------

juniper config ファイルを `.junos` という拡張子で保存して vim で開くか、
juniper config ファイルの先頭に以下行を追加して保存し vim で開くと色付けしてくれる。

```sh
# vim: set ft=junos:
```

もしくは、juniper config ファイルを vim 開いている時に `:set ft=junos` を実行する。

