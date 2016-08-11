---
layout: post
title: "ansible で MacBook Pro をセットアップ"
date: 2016-08-10 15:15
comments: true
categories: ansible mac
---

MacBook Pro 2700/13.3 MF839J/A を購入したので (わーい) 、構成管理ツール ansible を使ってセットアップした。

![macbook pro](/images/20160810_macbookpro.jpg)

t-wada さんのエントリ [Mac の開発環境構築を自動化する (2015 年初旬編) - t-wadaのブログ変更する](http://t-wada.hatenablog.jp/entry/mac-provisioning-by-ansible)
を見れば事足りると思われるので、本稿は自分用のメモ(t-wada さんのを少しだけカスタマイズ & 補足)。


<!-- more -->



1. 事前準備
===========


1.1. SSH
--------

ansible では、コントローラ (Control Machine) から SSH 越しで管理対象ノード (Managed Node) を操作する。
今回は Control Machine も Managed Node も同一マシン (Macbook) なので、localhost に SSH できるようにする。


まず SSH の許可設定。
Mac で「システム環境設定」 > 「共有」 > 「リモートログイン」にチェック。


次にパスなしログインするために公開鍵を設定する。
[こちらでも紹介したとおり](http://momota.github.io/blog/2016/02/08/ansible/)、以下のようにする。

```sh
# no-pass SSH key を生成
$ ssh-keygen -t rsa

# authorized_keysの登録
$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys

# SSH アクセス確認
$ ssh 127.0.0.1
```


1.2. Xcode
----------

Homebrew をインストールするために必要なので、App Store から XCode をインストールする。
XCode インストール後、ライセンス同意する。

```sh
$ sudo xcode build -license
```

ライセンス規約みたいな文章がつらつらと出てくるので、最後に「agree」と入力する。

XCode のコマンドラインツールもインストールする。

```sh
$ xcode-select --install
```


1.3. Homebrew
-------------

Homebrew は Mac 用のパッケージマネージャ (yum とか apt 的なやつ。
Homebrew って、ホームブリューと読んでいるけどホームブルーのほうが正しい発音ぽい)。

[公式サイト](http://brew.sh/index_ja.html)の案内に従い、以下のワンライナーで Homebrew をインストールする。

```sh
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

上記ワンライナーでインストールするとたぶん最新版がインストールされるが、インストール後 `brew doctor` で古いと出力されたらアップデートする。

```sh
$ brew doctor
$ brew update
```


1.4. Ansible
------------

Ansible に必要な python はすでに 2.7 系がプリインストールされていたので、
そのまま Homebrew で Ansible をインストールする。

```
$ brew install ansible
$ ansible --version
```



2. Ansible Playbook を開発する
==============================

前回の [Ansible でノート PC をセットアップ](http://momota.github.io/blog/2016/02/08/ansible/) で作った仕組みにのせる。

t-wada さんの場合、1 枚の Playbook に変数やタスクを記述していたが、ありものは活用したいという思いがあり。

今回は以下のようなアレンジをした。

- インストールするパッケージを列挙した変数を `vars/common.yml` に切り出す
```yaml
# for MacOSX
homebrew_taps:
  - homebrew/versions
  - homebrew/binary
  - homebrew/dupes
  - caskroom/cask
  - sanemat/font
homebrew_packages:
  - { name: readline }
  - { name: openssl }
  # (略)
homebrew_cask_packages:
  - { name: iterm2 }
  - { name: firefox }
  - { name: google-chrome }
  - { name: google-japanese-ime }
  # (略)
```
- CentOS などへもインストールするような共通のパッケージは `common/` 以下の role を利用する
  - 今回は tmux とzsh のみ
- Mac 固有のタスクは、`mac/` 以下に切り出す
  - brew (cask も) 用のrole を作る
  - ricty フォントインストールは個別の処理が多かったため別の role で切り出す
- インベントリファイル `hosts` に mac 用グループ `[mac]` を作る
```ini
[mac]
127.0.0.1
```

ディレクトリ構造は以下のようになった。

```
└── laptop-build
    ├── centos
    │   ├── docker
    │   │   ├── files
    │   │   └── tasks
    │   └── yum
    │       └── tasks
    ├── common
    │   ├── dotfiles
    │   │   ├── meta
    │   │   └── tasks
    │   ├── dstat
    │   │   └── tasks
    │   ├── guest_account
    │   │   └── tasks
    │   ├── ruby
    │   │   ├── meta
    │   │   └── tasks
    │   ├── tmux
    │   │   └── tasks
    │   ├── vim
    │   │   └── tasks
    │   └── zsh
    │       └── tasks
    ├── mac
    │   ├── brew
    │   │   └── tasks
    │   └── ricty
    │       ├── handlers
    │       └── tasks
    └── vars
```

できあがった Playbook は Github にあげた。 [momota/laptop-build](https://github.com/momota/laptop-build)

3. Ansible を実行する
=====================

3.1. ansible のログ出力設定
---------------------------

`tee` とかを使って Playbook の実行出力結果をファイルに保存しようと思ったが、ansible のログ機能を使った。
`ansible.cfg` に以下を記述。

```ini
[defaults]
log_path=/var/log/ansible.log
```

mac の場合、`/etc/ansible/ansible.cfg` がなかった (ディレクトリごとない) ので、`~/.ansible.cfg` を作成した。



3.2. ansible の実行
-------------------

absible-playbook コマンドで実行する。


```sh
# git clone
$ git clone https://github.com/momota/laptop-build
$ cd laptop-build

# execute playbook
$ HOMEBREW_CASK_OPTS="--appdir=/Applications" ansible-playbook -i hosts mac/site.yml -vv -K
```

`HOMEBREW_CASK_OPTS="--appdir=/Applications"` をつけないと、アプリケーションによって `/Applications` だったり、 `~/Applications` だったりにシンボリックリンクリンクが作られてしまうとのこと。

ログインシェルを zsh に変更するタスクで `sudo` するので、`-K` オプション付き。



4. Playbook のトラブルフォロー
==============================


4.1. shell モジュールでcommand not found
----------------------------------------

Playbook をコピペして実行したら、shell モジュールで `brew` を呼んでいるタスクから
`command not found` エラーが返ってきた。
フルパスで指定し直す。


4.2. cask パッケージ名の誤り
----------------------------

`/vars/common.yml` にインストールしたい cask パッケージ名を定義するが、パッケージ名を謝って途中で処理がこけた。
もちろんググっても良いと思うが、事前に以下のコマンドなどで調べるとよさそう。

```sh
$ brew cask search [探したいパッケージ]
```

4.3. cask インストールの処理停止
--------------------------------

Homebrew cask でパッケージインストール途中で処理が止まることがあった。(2, 3回)
どんなに待っても止まったままなので、 `Ctrl-c` で中断する。

ansible-playbook コマンドに `-vvv` オプションなどを付けるも、原因はよく分からず。
たぶん、ansible ではなく brew 側に原因がありそう。


インストール処理で止まっているパッケージは、ansible で表示されているパッケージの、`vars/common.yml` での変数定義上の次のパッケージ。
途中で止まっているパッケージもインストール一覧に出力される。

```sh
$ brew cask list
#=> インストール処理中のパッケージも出力される
```

リストには表示されるものの、中途半端にインストールされていそうなので、
`brew` コマンドでインストールし直して Playbook を再実行する。

```sh
$ brew cask install --force [途中で止まったパッケージ]

# アンインストールして、再インストールしても良いかも
#   $ brew cask uninstall [途中で止まったパッケージ]
#   $ brew cask install [途中で止まったパッケージ]

$ HOMEBREW_CASK_OPTS="--appdir=/Applications" ansible-playbook -i hosts mac/site.yml -vv -K
```

なにも考えずに re-run できるのはうれしい。 (これが冪等性のパワーか)

