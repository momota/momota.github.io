---
layout: post
title: "ansible ABC"
date: 2016-08-19 08:22
comments: true
categories: ansible
---
チーム内での Ansible 勉強会の資料を公開。


![ansible logo](/images/20160818_ansible-logo.png)


公式サイトは以下。

- http://www.ansible.com
- http://docs.ansible.com

前提知識としては、以下。知っていたほうが理解が早い。

- unix コマンド
- YAML
  - [ansible使いのためのYAML入門 - @znz blog](http://blog.n-z.jp/blog/2014-06-21-ansible-yaml.html)


公式から引用すると、Ansible は **simple IT automation engine**。

> Ansible is a radically **simple IT automation engine** that automates cloud provisioning, configuration management, application deployment, intra-service orchestration, and many other IT needs.
> 
> ---    [How Ansible Works](https://www.ansible.com/how-ansible-works) より


<!-- more -->

Ansible の概要
--------------

- Ansible は、サーバの構成管理ツールと言われている。
  - サーバ構成管理とは? Ansible が主に対象としているのは以下。
    - サーバのソフトウェアインストール/アップデート
    - 設定ファイルの作成/修正/削除
    - サービス停止/起動
    - ファイル (アプリケーション) の配布
  - 類似ツールに、Chef や Puppet がある。
  - Infrastructure as Code の流れや世の中のクラウド化に伴い、評価されている
  - Infrastructure as Code: インフラの状態をコード化できる
    - ソフトウェア開発でのナレッジが活かせる
      - バージョン管理: git, subversion, ...
      - コードレビュー: pull request, other tools, ...
      - テスト: test framework like serverspec, infrataster, ...
      - デプロイ、CI ... : jenkins, ... 
    - 設計ドキュメントと実装との差分がある程度埋まる
- もともとはAnsible, Inc により開発されていたが、2015年10月に Redhat により買収された

以下は、Ansible の動作イメージ。(https://sysadmincasts.com/episodes/46-configuration-management-with-ansible-part-3-4 より引用)


![diagram](/images/20160818_ansible_overview.gif)


Ansible の特徴・利点
====================

- 冪等性
  - 何回やっても同じ結果になること。
    - モジュール側でチェック処理などを吸収してくれる。
    - shell モジュールや rawモジュールなど、一部、冪等性が担保されないものもある。
  - 単純 re-run が可能。
  - Immutable Infrastructure との相性の良さ。
- Python の "バッテリー同梱 (Battery Included)" という考え方を引き継いでいる。
  - 標準モジュール/機能の豊富さ。
- Push 型アーキテクチャ
  - 管理対象ノード (Managed Node) に SSH さえできれば良い。
  - エージェントレス。
  - マルチプロセスで並列実行が可能。
  - Ansible 実行サーバを Control Machine, 管理対象ノードを Managed Nodeと呼ぶ。
- シンプル
  - 独自 DSL ではなく、YAML。
    - データ志向の自動化言語である YAML
    - 学習コストが低い。chef や puppet のように Ruby などの独自 DSL を覚える必要がない。
  - ファイル・ディレクトリ構造の自由度の高さ。決まり事が少ない。
  - Ansible 自体は Python で書かれているが、プラグインなどの開発が不要であれば、Python を使う必要なし。


Ansible の用途
==============

- 複数サーバへの……
  - バッチ処理
    - システム全体の設定 (`/etc` 以下の設定)
      - hosts へのエントリ
    - ネットワーク設定
      - ルーティングテーブルエントリの追加
      - SOCKS プロキシ
      - proxy.pac
    - ファイル配布
      - SSHキーの配布
  - ミドルウェアのインストール/アップデート
    - yum
    - apt
    - homebrew
    - gem
  - アプリケーションデプロイ


公式サイトの Use case を直訳すると、以下。
> - provisioning (Bootstrapping と呼ぶほうが正しいかも)
>   - ベアメタルサーバやVMに対する、PXEブート (遠隔電源操作), キックスタート (linux の自動インストール)
>   - VM (クラウドインスタンス) に対する、テンプレートからの作成
> - configuration management
>   - コンフィグファイルの中央管理
> - application deployment
>   - Ansible でアプリケーションを定義しデプロイすることで、開発環境から本番環境までのアプリケーション全体のライフサイクルを効果的に管理できる 
> - continuous delivery
>   - CI/CD のパイプラインはたくさんのチームから依頼を引き受けることになる。だれもが使えるシンプルなAnsible を使うことで適切にアプリケーションのデプロイ管理ができる
> - security & compliance
>   - Ansible でセキュリティポリシを定義することで、サイト全体のセキュリティポリシのスキャンと修復が可能になる。
> - orchestration
>   - 管理対象のコンフィグは多種多様に存在し、かつ、それぞれが相互に作用している。Ansible は、この複雑で混沌とした中に秩序をもたらす。


仕事ではあまり OS を扱うことは少なく、以下のようなレイヤの低いやつを raw モジュールを
駆使して構成変更したり情報取得したりしている。

- ESXi
- iLO
- BIOS
- その他 SSH 可能なノード


Ansible を動かしてみよう
========================

Ansible のインストール
----------------------

[Installation - Ansible Documentation](http://docs.ansible.com/ansible/intro_installation.html#getting-ansible) を参照

1. Control Machine : Ansible 実行サーバ
    - Python 2.6 or 2.7 が必要
    - yum や rpm, apt などのパッケージ管理システムでインストール可能
2. Managed Node : 管理対象ノード
    - Control Machine から SSH できればOK

Ansible 実行してみる
--------------------

ansile を単体実行するコマンド書式は以下。

```
$ ansible <host-pattern> [-f forks] [-m module_name] [-a args]
```

オプション | 意味
-----------|------------------------------------------------
-f         | 実行多重度。デフォルトは5 (ansible.cfg で規定)
-m         | モジュール名
-a         | モジュール引数

[モジュールはこちらを参照。](http://docs.ansible.com/ansible/modules_by_category.html)

以下はローカルで動くモジュールの一部を紹介。

モジュール | 機能
-----------|------------------------------------------------
ping       | その名の通り ping
shell      | コマンドを引数に渡して shell 実行可能
file       | ファイルのパーミション設定とか、ディレクトリ作成とか
template   | Control Machine のテンプレートファイル (テンプレートエンジンは jinja2 ) を Managed Node に配布
service    | init system のサービスを制御
sysctrl    | カーネルパラメータの操作
raw        | ssh でナマの文字列をそのまま流し込む

インターネットに接続していれば、以下も有用。

モジュール | 機能
-----------|------------------------------------------------
yum        | yum によるソフトウェアパッケージ管理
get_url    | HTTP でのダウンロード (curl, wget 的な)
git        | git リポジトリからの clone

バージョンが 2 系からJunos とか Cisco, ESXi 用のモジュールもある。


`<host-pattern>` でホストを指定する。
all とすると、`/etc/ansible/hosts` で定義しているすべてのホストが対象となる。


以下は `ping` モジュールを利用したコマンド。
localhost に ping する ansible コマンド。
`-m` オプションのあとに `ping` モジュールを指定する。

```sh
$ ansible localhost -m ping
localhost | success >> {
    "changed": false,
    "ping": "pong"
}
```

以下は、`shell` モジュールを利用したコマンド。
`-a` オプションのあとにモジュール引数を渡す。

```sh
$ ansible localhost -m shell -a "uname -a; date; id; ifconfig"
localhost | success | rc=0 >>
Linux m 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:12 UTC 2014 i686 i686 i686 GNU/Linux
Thu Aug  4 17:37:49 JST 2016
uid=1000(momota) gid=1000(momota) groups=1000(momota),4(adm),24(cdrom),27(sudo),30(dip),33(www-data),46(plugdev),108(lpadmin),124(sambashare)
docker0   Link encap:Ethernet  HWaddr 56:84:7a:fe:97:99
          inet addr:172.*.*.*  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

eth0      Link encap:Ethernet  HWaddr 00:1e:c9:80:66:c6
          inet addr:172.*.*.*  Bcast:172.104.43.255  Mask:255.255.254.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:40670341 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1099217 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:4490922632 (4.4 GB)  TX bytes:329019625 (329.0 MB)
          Interrupt:21 Memory:febe0000-fec00000

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:47797 errors:0 dropped:0 overruns:0 frame:0
          TX packets:47797 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:5233688 (5.2 MB)  TX bytes:5233688 (5.2 MB)
```

ただし、実際には `ansible` コマンドを使うことはめったにない。

複数のノードに対してタスクを実行したり、複雑なタスクを実行する場合は、
Inventory ファイルや Playbook ファイルを利用する。

Ansible の主なファイルは以下。

- ansible.cfg
  - ansible に関するデフォルト設定ファイル。`/etc/ansible/ansible.cfg`
- Inventory
  - ansible で管理対象ノードを定義するファイル
- Playbook
  - タスク (実行手順) を定義するファイル


Inventory
=========

Inventory ファイルは、ansible 用の hosts ファイル。
ホストやグルーピングの設定、変数などの定義が可能。
ホストや IP アドレスはレンジでの指定も可能。

- ホストとグループの定義
- ホスト変数とグループ変数
- グループのグループ

以下は、デフォルト設定 `/etc/ansible/hosts`。

```ini
#   - Comments begin with the '#' character
#   - Blank lines are ignored
#   - Groups of hosts are delimited by [header] elements
#   - You can enter hostnames or ip addresses
#   - A hostname/ip can be a member of multiple groups

# Ex 1: Ungrouped hosts, specify before any group headers.

green.example.com
blue.example.com
192.168.100.1
192.168.100.10

# Ex 2: A collection of hosts belonging to the 'webservers' group

[webservers]
alpha.example.org
beta.example.org
192.168.1.100
192.168.1.110

# If you have multiple hosts following a pattern you can specify
# them like this:

www[001:006].example.com

# Ex 3: A collection of database servers in the 'dbservers' group

[dbservers]

db01.intranet.mydomain.net
db02.intranet.mydomain.net
10.25.1.56
10.25.1.57

# Here's another example of host ranges, this time there are no
# leading 0s:

db-[99:101]-node.example.com
```

Playbook
========

Playbook はタスクをまとめて、1つのファイルに記述したもの。
実行順序や依存関係、変数を利用した処理、ループ処理などが表現できる。

Playbookの基本的なフォーマットは以下。

```yaml
- hosts: <対象とするホスト/グループ>
  var:
    <変数名1>: <値>
    <変数名2>: <値>
      :
      :
  remote_user: <処理を実行するユーザー名>
  become: yes   # sudo するか
  tasks:
    <タスク1>
    <タスク2>
      :
      :
  handlers:
    <ハンドラ1>
    <ハンドラ2>
      :
      :
```

タスク
------

ansible ではホストに対する処理をタスクという単位で管理する。
通常、1つのタスクには1つのモジュールを指定する。

```yaml
  - name: <タスクの説明>
    <モジュール名>: <実行時に与えるパラメータ>
```

name は description なので省略可能だが、メンテナンス性や playbook 実行時にわかりやすいので絶対書いたほうが良い。

name は日本語でも記載可能。 (文字コードは注意)


ハンドラ
--------

ハンドラは、特定のタスクの実行後に、予め指定した処理を実行するための仕組み。
以下のように記述する。

```yaml
tasks:
  - name: be sure httpd is installed
    yum: name=httpd state=installed
    notify:
      - restart httpd
handlers:
  - name: restart httpd
    service: name=httpd state=restarted
```

この場合、「be sure httpd is installed」というタスクが実行されると、続いて「notify」で指定された「restart httpd」というハンドラが実行される。


ループ
------

`with_items` で列挙した項目に対してループ処理が可能。

[Loops - Ansible Documentation](http://docs.ansible.com/ansible/playbooks_loops.html)

リモートのコマンド結果 (標準出力)に対して、一行ずつ処理するタスクが以下。

{% raw %}
```yaml
- name: "リモートのコマンド結果を使ったループの例"
  shell: /usr/bin/something
  register: command_result

- name: "結果を受け取って何かするタスク"
  shell: /usr/bin/something_else --param {{ item }}
  with_items: "{{ command_result.stdout_lines }}"
```
{% endraw %}

こんな感じで直接リストを書いてもOK。

{% raw %}
```yaml
- name: "リストからパッケージをインストールしていく"
  yum: name={{ item }} state=latest
  with_items:
    - httpd
    - zsh
    - tmux
    - vim
```
{% endraw %}



playbook のサンプル
===================

まずはインベントリファイル `sample_hosts` を作成する

"fujiko" グループに所属しているホストのリスト (計3台分) を記述している。

```ini
[fujiko]
doraemon
nobita
shizuka
```

次にplaybook ファイル `sample_playbook.yml` を作成する。

計 3 つのタスクを記述している。

`register: <変数名1>` という書式でタスク実行結果を変数に保存する。 この変数を Registered Variable と呼ぶ。

`debug: var=<変数名1>/stdout_lines` により、変数に保存したタスク実行結果 (の標準出力) を表示している。

```yaml
---
- hosts: fujiko
  tasks:
    - name: "死活"
      ping:

    - name: "ホスト名"
      shell:    hostname
      register: host

    - debug: var=host.stdout_lines
      when: host | success

    - name: "ユーザ名"
      shell:    whoami
      register: user

    - debug: var=user.stdout_lines
      when: user | success
```

playbook の実行前に syntax check (構文の確認)をする。

```sh
$ ansible-playbook -i sample_hosts sample_playbook.yml --syntax-check

playbook: sample-playbook.yml

```


次に playbook を dry-run。
実行する前に何が変更されるか、特に、実行時に消えてしまうリソースや入れ替わるリソースを知りたい場合に有効。

```sh
$ ansible-playbook -i sample_hosts sample_playbook.yml --check
--- snip ---
```

Playbook を実行する。`--step` オプションを付けると、タスク単位にステップ実行もできる。

```sh
$ ansible-playbook -i sample_hosts sample_playbook.yml

PLAY [fujiko] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [doraemon]
ok: [nobita]
ok: [shizuka]

TASK: [死活確認] **********************************************************
ok: [nobita]
ok: [doraemon]
ok: [shizuka]

TASK: [ホスト名] **********************************************************
changed: [nobita]
changed: [doraemon]
changed: [shizuka]

TASK: [debug var=host.stdout_lines] *******************************************
ok: [nobita] => {
    "var": {
        "host.stdout_lines": [
            "nobita"
        ]
    }
}
ok: [doraemon] => {
    "var": {
        "host.stdout_lines": [
            "doraemon"
        ]
    }
}
ok: [shizuka] => {
    "var": {
        "host.stdout_lines": [
            "shizuka"
        ]
    }
}

TASK: [ユーザ名] **********************************************************
changed: [nobita]
changed: [doraemon]
changed: [shizuka]

TASK: [debug var=user.stdout_lines] *******************************************
ok: [nobita] => {
    "var": {
        "user.stdout_lines": [
            "momota"
        ]
    }
}
ok: [doraemon] => {
    "var": {
        "user.stdout_lines": [
            "momota"
        ]
    }
}
ok: [shizuka] => {
    "var": {
        "user.stdout_lines": [
            "momota"
        ]
    }
}

PLAY RECAP ********************************************************************
doraemon                   : ok=6    changed=2    unreachable=0    failed=0
nobita                     : ok=6    changed=2    unreachable=0    failed=0
shizuka                    : ok=6    changed=2    unreachable=0    failed=0

```

playbook は role という単位で分割可能。再利用性が高くなり、1 つ 1 つの playbook の
見通しも良くなるなるので、相互に関係のないタスクは分離していくほうが望ましい。

大規模な環境用に、[Best practices](http://docs.ansible.com/ansible/playbooks_best_practices.html) が示されているので、ご参考に。


システム情報を利用する: Facts
=============================

Playbook 実行結果に「GATHERING FACTS」と出力されている。
これは対象サーバから情報を収集する処理。たとえば、リモートサーバのIPアドレスや
OS名などのシステム情報を、Playbook 中で変数として扱うことができる。

次のように `setup` モジュールにより、どんな変数が利用可能かを確認することができる。

```json
$ ansible some-host -m setup
"ansible_all_ipv4_addresses": [
    "REDACTED IP ADDRESS"
],
"ansible_all_ipv6_addresses": [
    "REDACTED IPV6 ADDRESS"
],
"ansible_architecture": "x86_64",
"ansible_bios_date": "09/20/2012",
"ansible_bios_version": "6.00",
"ansible_cmdline": {
    "BOOT_IMAGE": "/boot/vmlinuz-3.5.0-23-generic",
    "quiet": true,
    "ro": true,
    "root": "UUID=4195bff4-e157-4e41-8701-e93f0aec9e22",
    "splash": true
},
"ansible_date_time": {
    "date": "2013-10-02",
    "day": "02",
    "epoch": "1380756810",
    "hour": "19",
    "iso8601": "2013-10-02T23:33:30Z",
    "iso8601_micro": "2013-10-02T23:33:30.036070Z",
    "minute": "33",
    "month": "10",
    "second": "30",
    "time": "19:33:30",
    "tz": "EDT",
    "year": "2013"
},
"ansible_default_ipv4": {
    "address": "REDACTED",
    "alias": "eth0",
    "gateway": "REDACTED",
    "interface": "eth0",
    "macaddress": "REDACTED",
    "mtu": 1500,
    "netmask": "255.255.255.0",
    "network": "REDACTED",
    "type": "ether"
},
"ansible_default_ipv6": {},
"ansible_devices": {
    "fd0": {
        "holders": [],
        "host": "",
        "model": null,
        "partitions": {},
        "removable": "1",
        "rotational": "1",
        "scheduler_mode": "deadline",
        "sectors": "0",
        "sectorsize": "512",
        "size": "0.00 Bytes",
        "support_discard": "0",
        "vendor": null
    },
    "sda": {
        "holders": [],
        "host": "SCSI storage controller: LSI Logic / Symbios Logic 53c1030 PCI-X Fusion-MPT Dual Ultra320 SCSI (rev 01)",
        "model": "VMware Virtual S",
        "partitions": {
            "sda1": {
                "sectors": "39843840",
                "sectorsize": 512,
                "size": "19.00 GB",
                "start": "2048"
            },
            "sda2": {
                "sectors": "2",
                "sectorsize": 512,
                "size": "1.00 KB",
                "start": "39847934"
            },
            "sda5": {
                "sectors": "2093056",
                "sectorsize": 512,
                "size": "1022.00 MB",
                "start": "39847936"
            }
        },
        "removable": "0",
        "rotational": "1",
        "scheduler_mode": "deadline",
        "sectors": "41943040",
        "sectorsize": "512",
        "size": "20.00 GB",
        "support_discard": "0",
        "vendor": "VMware,"
    },
    "sr0": {
        "holders": [],
        "host": "IDE interface: Intel Corporation 82371AB/EB/MB PIIX4 IDE (rev 01)",
        "model": "VMware IDE CDR10",
        "partitions": {},
        "removable": "1",
        "rotational": "1",
        "scheduler_mode": "deadline",
        "sectors": "2097151",
        "sectorsize": "512",
        "size": "1024.00 MB",
        "support_discard": "0",
        "vendor": "NECVMWar"
    }
},
"ansible_distribution": "Ubuntu",
"ansible_distribution_release": "precise",
"ansible_distribution_version": "12.04",
"ansible_domain": "",
"ansible_env": {
    "COLORTERM": "gnome-terminal",
    "DISPLAY": ":0",
    "HOME": "/home/mdehaan",
    "LANG": "C",
    "LESSCLOSE": "/usr/bin/lesspipe %s %s",
    "LESSOPEN": "| /usr/bin/lesspipe %s",
    "LOGNAME": "root",
    "LS_COLORS": "rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arj=01;31:*.taz=01;31:*.lzh=01;31:*.lzma=01;31:*.tlz=01;31:*.txz=01;31:*.zip=01;31:*.z=01;31:*.Z=01;31:*.dz=01;31:*.gz=01;31:*.lz=01;31:*.xz=01;31:*.bz2=01;31:*.bz=01;31:*.tbz=01;31:*.tbz2=01;31:*.tz=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.war=01;31:*.ear=01;31:*.sar=01;31:*.rar=01;31:*.ace=01;31:*.zoo=01;31:*.cpio=01;31:*.7z=01;31:*.rz=01;31:*.jpg=01;35:*.jpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.webm=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.axv=01;35:*.anx=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=00;36:*.au=00;36:*.flac=00;36:*.mid=00;36:*.midi=00;36:*.mka=00;36:*.mp3=00;36:*.mpc=00;36:*.ogg=00;36:*.ra=00;36:*.wav=00;36:*.axa=00;36:*.oga=00;36:*.spx=00;36:*.xspf=00;36:",
    "MAIL": "/var/mail/root",
    "OLDPWD": "/root/ansible/docsite",
    "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "PWD": "/root/ansible",
    "SHELL": "/bin/bash",
    "SHLVL": "1",
    "SUDO_COMMAND": "/bin/bash",
    "SUDO_GID": "1000",
    "SUDO_UID": "1000",
    "SUDO_USER": "mdehaan",
    "TERM": "xterm",
    "USER": "root",
    "USERNAME": "root",
    "XAUTHORITY": "/home/mdehaan/.Xauthority",
    "_": "/usr/local/bin/ansible"
},
"ansible_eth0": {
    "active": true,
    "device": "eth0",
    "ipv4": {
        "address": "REDACTED",
        "netmask": "255.255.255.0",
        "network": "REDACTED"
    },
    "ipv6": [
        {
            "address": "REDACTED",
            "prefix": "64",
            "scope": "link"
        }
    ],
    "macaddress": "REDACTED",
    "module": "e1000",
    "mtu": 1500,
    "type": "ether"
},
"ansible_form_factor": "Other",
"ansible_fqdn": "ubuntu2.example.com",
"ansible_hostname": "ubuntu2",
"ansible_interfaces": [
    "lo",
    "eth0"
],
"ansible_kernel": "3.5.0-23-generic",
"ansible_lo": {
    "active": true,
    "device": "lo",
    "ipv4": {
        "address": "127.0.0.1",
        "netmask": "255.0.0.0",
        "network": "127.0.0.0"
    },
    "ipv6": [
        {
            "address": "::1",
            "prefix": "128",
            "scope": "host"
        }
    ],
    "mtu": 16436,
    "type": "loopback"
},
"ansible_lsb": {
    "codename": "precise",
    "description": "Ubuntu 12.04.2 LTS",
    "id": "Ubuntu",
    "major_release": "12",
    "release": "12.04"
},
"ansible_machine": "x86_64",
"ansible_memfree_mb": 74,
"ansible_memtotal_mb": 991,
"ansible_mounts": [
    {
        "device": "/dev/sda1",
        "fstype": "ext4",
        "mount": "/",
        "options": "rw,errors=remount-ro",
        "size_available": 15032406016,
        "size_total": 20079898624
    }
],
"ansible_nodename": "ubuntu2.example.com",
"ansible_os_family": "Debian",
"ansible_pkg_mgr": "apt",
"ansible_processor": [
    "Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz"
],
"ansible_processor_cores": 1,
"ansible_processor_count": 1,
"ansible_processor_threads_per_core": 1,
"ansible_processor_vcpus": 1,
"ansible_product_name": "VMware Virtual Platform",
"ansible_product_serial": "REDACTED",
"ansible_product_uuid": "REDACTED",
"ansible_product_version": "None",
"ansible_python_version": "2.7.3",
"ansible_selinux": false,
"ansible_ssh_host_key_dsa_public": "REDACTED KEY VALUE"
"ansible_ssh_host_key_ecdsa_public": "REDACTED KEY VALUE"
"ansible_ssh_host_key_rsa_public": "REDACTED KEY VALUE"
"ansible_swapfree_mb": 665,
"ansible_swaptotal_mb": 1021,
"ansible_system": "Linux",
"ansible_system_vendor": "VMware, Inc.",
"ansible_user_id": "root",
"ansible_userspace_architecture": "x86_64",
"ansible_userspace_bits": "64",
"ansible_virtualization_role": "guest",
"ansible_virtualization_type": "VMware"
```

たとえば、IPアドレスは {% raw %}`{{ ansible_eth0["ipv4"]["address"] }}` {% endraw %}のように、Playbook の中で変数として
扱うことができる。

その他、`facter` や `ohai` がリモートホストにインストールされていればそれらも利用可能。
facts よりも詳細な情報が取得できる。

この facts は、いまのところリモートホストが Linux でのみ動作する。
リモートホストが windows やネットワーク機器の場合は以下のように無効化できる。

```yaml
- hosts: whatever
  gather_facts: no
```

また、システム情報を利用しない場合も無効化することで実行時間短縮が図れる。


links
=====

- [Ansible Tutorial](http(//yteraoka.github.io/ansible-tutorial/)
- [ansible playbook コマンド オプション](https(//github.com/yteraoka/ansible-tutorial/wiki/ansible-playbook%20%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89)
- [Ansible Performance Tuning (for Fun and Profit)](https(//www.ansible.com/blog/ansible-performance-tuning)
- [Ansibleのアーキテクチャー( 構成管理を超えて](http(//tdoc.info/blog/2014/01/20/ansible_beyond_configuration.html)
- [Ansible オレオレベストプラクティス](http(//qiita.com/yteraoka/items/5ed2bddefff32e1b9faf)
