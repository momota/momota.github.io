---
layout: post
title: "CentOS 7 を USB メモリからインストール"
date: 2015-11-15 21:15
comments: true
categories: centos mac
---


新しい PC が来たので、もともと使ってた東芝 Dynabook R731/B (windows 7)に CentOS7 をインストールした。
今までは DVD-R に ISO を焼いていたけど、自前の PC の DVD ドライブが壊れていたので
USB メモリを使ってインストールしてみた。

<!-- more -->

1. USB メモリをフォーマットする
-------------------------------

MAC で USB メモリをフォーマット (初期化) しておく。

1. ディスクユーティリティを起動する (アプリケーション > ディスクユーティリティ)
1. 左側のペインからフォーマットしたいデバイスを選択し、`消去` タブをクリックする
  - 容量や、ドライブ名などから判断
1. フォーマット形式は MS-DOS (FAT) を選択し、消去をクリックする


2. インストールイメージをダウンロードする
-----------------------------------------

[CentOS7 の DVD ISO をダウンロードする。](https://www.centos.org/download/)

[md5 ハッシュもチェックしておく。](http://ftp.jaist.ac.jp/pub/Linux/CentOS/7/isos/x86_64/md5sum.txt)


ダウンロードされた ISO イメージが正しくダウンロードされていることを確認する。(linux だと `md5sum` コマンド)

```
$ md5 CentOS-7-x86_64-DVD-1503-01.iso | grep 99e450fb1b22d2e528757653fcbf5fdc
```


3. USB メモリにインストールイメージを書き込む
---------------------------------------------

USB メモリにインストールイメージを書き込む
ターミナルから.iso を .img へ変換する。

```
$ hdiutil convert -format UDRW -o centos7.img DOWNLOADED_ISO.iso
#=> centos7.img.dmg が出力される
$ mv centos7.img.dmg centos7.img
```

USB メモリをアンマウントする。

```
# USB メモリがどう認識されているかを確認する (/dev/diskXX)
$ diskutil list
# USB メモリをアンマウントする (/dev/disk1の場合)
$ diskutil unMountDisk /dev/disk1
$ diskutil list
```

dd で USB メモリにインストールイメージを書き込む。

```
$ sudo dd if=hoge.img of=/dev/disk1 bs=1m
$ echo $?
#=>0
```

書き込みは結構時間がかかる。
完了後、「この Mac では読めない形式だ」的なダイアログが出るが無視。

USB メモリをイジェクトして取り外す。

```
$ sudo diskutil eject /dev/disk1
```


4. ノート PC にインストールする
-------------------------------

※ USB メモリをノートPC側で使える前提。USBメモリのドライバがインストールされた状態からスタート。

BIOS設定画面から起動デバイスの優先順位を変更する。
USB デバイスを最優先にしておく。

インストールイメージを焼いた USB メモリを刺したままノート PC を起動してみて「CentOSをインストール？それともお試しインストール？」みたいな画面が出てくる。
お試しインストールしてみて、キーボードやタッチパッドが使えることを確認する。
確認ができたら再起動し、インストールする。


