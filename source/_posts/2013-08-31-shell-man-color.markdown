---
layout: post
title: "man ページに色付けする (zsh, bash)"
date: 2013-08-31 22:31
comments: true
categories: zsh bash linux
---
# man コマンドで表示するマニュアルページに色付けする

man コマンド、man コマンド、マンコマンド… 連呼している私はもう今年で31歳です。こんばんわ。

`~/.zshrc` [bash 使いなら `~/.bash_profile` ] に以下を追記するとman に色がついて読みやすいっす。


<!-- more -->

```sh
man() {
        env \
                LESS_TERMCAP_mb=$(printf "\e[1;31m") \
                LESS_TERMCAP_md=$(printf "\e[1;31m") \
                LESS_TERMCAP_me=$(printf "\e[0m") \
                LESS_TERMCAP_se=$(printf "\e[0m") \
                LESS_TERMCAP_so=$(printf "\e[1;44;33m") \
                LESS_TERMCAP_ue=$(printf "\e[0m") \
                LESS_TERMCAP_us=$(printf "\e[1;32m") \
                man "$@"
}
```

- before

![before_man](https://dl.dropboxusercontent.com/u/28495046/octopress/20130831_sh/sh_before.png)

- after

![after_man](https://dl.dropboxusercontent.com/u/28495046/octopress/20130831_sh/sh_after.png)

