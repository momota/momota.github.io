---
layout: post
title: "zshプラグイン: Incremental completion for zsh"
date: 2019-03-29 20:12
comments: true
categories: zsh
---

zsh でインクリメンタルな補完を行うため、[Incremental completion on zsh](https://mimosa-pudica.net/zsh-incremental.html) を導入した。

こんな感じで便利。

![](/images/20190329_incremental-completion-for-zsh/incr-zsh.gif)


<!-- more -->

[robbyrussell/oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh) を導入している前提。

oh-my-zsh の導入に関しては、こちらの記事を参照: [zsh をログインシェルにしてワーク・ライフ・バランスを改善する (oh-my-zsh) - momota.txt](http://momota.github.io/blog/2013/09/07/oh-my-zsh/)


今回利用するカスタムスクリプトは `~/.oh-my-zsh/custom/` 以下に置けばログイン時にロードしてくれる。したがって、公式に書いてある `source incr*.zsh` を `.zshrc` へ書く必要はない。

スクリプト名にバージョン名が入っているのが気持ち悪かったので、`incremental_completion_for_zsh.zsh` へ修正した。

```sh
$ curl -o  ~/.oh-my-zsh/custom/incremental_completion_for_zsh.zsh \
> https://mimosa-pudica.net/src/incr-0.2.zsh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2631  100  2631    0     0   2386      0  0:00:01  0:00:01 --:--:--  2389
```

サジェストされるコマンド文字の色を控えめにしたかったので、以下の修正を加えた。
(グリーンからグレーへ変更) 

```diff
@@ -82,7 +82,7 @@
                        BUFFER="$buffer_org"
                        CURSOR="$cursor_org"
                fi
-               echo -n "\e[32m"
+               echo -n "\e[1;30m"
        else
                zle -M ""
        fi
```
