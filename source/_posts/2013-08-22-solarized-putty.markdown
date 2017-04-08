---
layout: post
title: "PuTTY へsolarized導入"
date: 2013-08-22 23:00
comments: true
categories: windows putty solarized
---
![solarized](http://ethanschoonover.com/solarized/img/solarized-yinyang.png)

人気のカラースキーム [Solarized](http://ethanschoonover.com/solarized) を、PuTTYへ適用する。
ちなみにパティと発音するみたいですね。ずっとプティと発音してました。

適用イメージは以下。

* before

![putty before](/images/20130822_putty_solarized/01_before_putty.png)

* after

![putty after](/images/20130822_putty_solarized/02_after_putty.png)


<!-- more -->

# 環境
* windows 7 professional service pack 1
* PuTTY 0.60-JP_Y-2007-08-06
* ちなみにフォントは、"Osakaー等幅" を利用


# solarized_dark.reg (solarized_light.reg)をダウンロードする

https://github.com/brantb/solarized/tree/master/putty-colors-solarized

solarized_dark.regをダウンロードし、`C:¥Program Files (x86)¥PuTTY¥` に保存


# solarized_dark.regを修正する

http://27213143.at.webry.info/201304/article_1.html
    
solarized_dark.regをテキストエディタで開き、ファイル中の `Solarized%20Dark(Solarized%20Light)` 部分をputtyのセッション名に修正する。下の例では セッション名は `_vagrant_ubuntu`。

![putty session name](/images/20130822_putty_solarized/putty_session_name.png)


# putty設定をインポートする
以下の2通りのやり方がある。お好きな方でどうぞ。

## (1) ダブルクリックでインポート
solarized_dark.reg をダブルクリックする。
     
## (2) コマンドプロンプトからインポート
http://103px.blog.fc2.com/blog-entry-46.html
     
windowsコマンドプロンプトから.regファイルをインポートする
```bat
     > reg import C:¥Program Files (x86)¥PuTTY¥solarized_dark.reg
```

