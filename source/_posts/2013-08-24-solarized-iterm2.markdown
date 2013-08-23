---
layout: post
title: "iTerm2へsolarized導入"
date: 2013-08-24 00:52
comments: true
categories: solarized, iterm2
---
![solarized](http://ethanschoonover.com/solarized/img/solarized-yinyang.png)

人気のカラースキーム [Solarized](http://ethanschoonover.com/solarized) を、iTerm2へ適用する。


適用イメージは以下。

* before

![iterm before](https://dl.dropboxusercontent.com/u/28495046/octopress/20130823_iterm_solarized/before_iterm.png)


* after

![iterm after](https://dl.dropboxusercontent.com/u/28495046/octopress/20130823_iterm_solarized/after_iterm.png)


<!-- more -->

1. Solarized*.itermcolors をダウンロードする

    https://github.com/altercation/solarized/tree/master/iterm2-colors-solarized からSolarized*.itermcolors ファイルをダウンロードする


2. iterm2 を開く

    ちなみにわたしは、`ctrl + space` でSpotlightを開き、`ite`入力くらいで起動しています。


3. preferencesダイアログを開く

    `command + o` でprofilesダイアログを開いて、`Edit Profiles...` ボタンを押下する。


4. solarized用のprofileを作成する

    preferences ダイアログの`Profiles`タブから、左下の方の`+`を押下し、プロファイルを追加する。
    `General`タブのBasicのNameを`solarized`とかに修正する


5. 1. でダウンロードしたファイルを読み込ませる

    `Colors` タブから `Load Presets...` ボタンを押下後、`import` を押す。1. でダウンロードしたSolarized*.itermcolorsファイルを選択する。

