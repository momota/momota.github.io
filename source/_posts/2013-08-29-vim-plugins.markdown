---
layout: post
title: "vim プラグインで幸せな生活を送る"
date: 2013-08-29 00:38
comments: true
categories: vim plugin
---
vimはhjklな変態的キーバインドから敬遠されがちだと思うけど強力なプラグインがあってこそ、やめられないとまらないわけであって、ここではプラグインの導入方法や管理方法を記載します。

<!-- more -->


## ノーマルなプラグイン導入

1. プラグインファイルをダウンロードしてくる

2. ダウンロードしたプラグインファイル一式を runtimepathで指定されたディレクトリ以下に放り込む

    - 通常は `$HOME/.vim` 。runtimepathは以下のviコマンドで確認できる。

```vim
:set runtimepath
```

helpを見てみると、以下ディレクトリ(優先順)がデフォルト設定されている。(vim74-kaoriya-win64で確認)

```vim
 :help runtimepath
 # => 以下抜粋
 'runtimepath' 'rtp' 文字列  (既定値:
                 Unix: "$HOME/.vim,
                 $VIM/vimfiles,
                 $VIMRUNTIME,
                 $VIM/vimfiles/after,
                 $HOME/.vim/after"
                 DOS, MS-Win系, OS/2: "$HOME/vimfiles,
                 $VIM/vimfiles,
                 $VIMRUNTIME,
                 $VIM/vimfiles/after,
                 $HOME/vimfiles/after"
                 Macintosh: "$VIM:vimfiles,
                 $VIMRUNTIME,
                 $VIM:vimfiles:after"
                 )

```

## プラグインによるプラグインパッケージ管理

上述したプラグイン導入を続けていくと、`$HOME/.vim/` 以下が散らかっていくことが容易に想像できるでしょう。散らかるだけで実害がなければ良いのですが、あるプラグインだけ削除したいときなどに難易度高くて困ります。
そこで、プラグインの相互独立性を担保しながら管理できるプラグインを導入します。

代表的なプラグインパッケージ管理プラグインは、以下です。

|プラグイン名 |概要       |
|-------------|-----------|
|[pathogen](https://github.com/tpope/vim-pathogen)      | `~/.vim/bundle/<プラグイン名>/` 以下の各ディレクトリも `~/.vim/` 直下と同じように読み込むようにするプラグイン。これにより、`~/.vim/bundle/` 以下にプラグインごとに別のディレクトリを切って管理をすることができるようになる。
|[vundle](https://github.com/gmarik/vundle)             |Vundle は Rails 3 で採用されている、Gem 管理システム Bundler に影響を受けたプラグイン管理システム。自分で `~/.vim/bundle` git cloneなどで放り込まなくても、vimコマンドでプラグイン追加が可能。
|[neobundle](https://github.com/Shougo/neobundle.vim)   |Vundleにインスパイアされたプラグイン管理システム。Vundleの改良版。

上記よりNeoBundleを選びたいと思います。



## NeoBundleインストールと設定

### 1. NeoBundleをセットアップする

- githubから最新NeoBundleのソースを clone する。ここでは、runtimepathを`~./.vim/`とします。


```sh
$ mkdir -p ~/.vim/bundle
$ git clone git://github.com/Shougo/neobundle.vim ~/.vim/bundle/neobundle.vim

```
### 2. NeoBundleを設定する

- `~/.vimrc` にNeoBundleの設定を加える。

```vim
" ----------------------------------------------------------------------------------------
"   neobundle
" ----------------------------------------------------------------------------------------
set nocompatible               " Be iMproved

if has('vim_starting')
set runtimepath+=~/.vim/bundle/neobundle.vim/
endif

call neobundle#rc(expand('~/.vim/bundle/'))

" Let NeoBundle manage NeoBundle
NeoBundleFetch 'Shougo/neobundle.vim'

" Recommended to install
" After install, turn shell ~/.vim/bundle/vimproc, (n,g)make -f your_machines_makefile
NeoBundle 'Shougo/vimproc', {
        \ 'build' : {
                \ 'windows' : 'make -f make_mingw32.mak',
                \ 'cygwin' : 'make -f make_cygwin.mak',
                \ 'mac' : 'make -f make_mac.mak',
                \ 'unix' : 'make -f make_unix.mak',
        \ },
\ }

filetype plugin indent on     " Required!

" Brief help
" :NeoBundleList          - list configured bundles
" :NeoBundleInstall(!)    - install(update) bundles
" :NeoBundleClean(!)      - confirm(or auto-approve) removal of unused bundles

" Installation check.
NeoBundleCheck
```

### 3. プラグインの取得

- `~/.vimrc`にインストールするプラグインを列挙する。(以下は例)
- 保存したあと、vim を起動し`:NeoBundleInstall` を実行するとプラグインがインストールされる。プラグインファイルは、`.vim/bundle/`以下にプラグイン別に保存される。もちろん各プラグインの設定は個別に `.vimrc` へ記述する必要がある。

```vim
" GitHubリポジトリにあるプラグインを利用する
" --> NeoBundle 'USER/REPOSITORY-NAME'
NeoBundle 'Shougo/neocomplcache'
NeoBundle 'Shougo/neosnippet'
NeoBundle 'Shougo/unite.vim'
NeoBundle 'thinca/vim-quickrun'
NeoBundle 'davidoc/taskpaper.vim'
NeoBundle 'itchyny/lightline.vim'
NeoBundle 'altercation/vim-colors-solarized'

"GitHub以外のGitリポジトリにあるプラグインを利用する
NeoBundle 'git://git.wincent.com/command-t.git'

" vim-scripts リポジトリにあるプラグインを利用する
NeoBundle 'surround.vim'

"Git以外のリポジトリにあるプラグインを利用する
NeoBundle 'http://svn.macports.org/repository/macports/contrib/mpvim/'
NeoBundle 'https://bitbucket.org/ns9tks/vim-fuzzyfinder'
```

- プラグインをアンインストールしたい場合は、`.vimrc`から当該プラグインの記述を消し、`:NeoBundleClean`を実行する。


## おすすめプラグイン

|プラグイン名  |概要                                        |
|--------------|--------------------------------------------|
|[neocomplcache](https://github.com/Shougo/neocomplcache.vim) |入力補完機能を提供する
|[neosnippet](https://github.com/Shougo/neosnippet.vim)    |スニペット入力サポート
|[lightline.vim](https://github.com/itchyny/lightline.vim) |ステータスライン表示をおしゃれに
|[taskpaper.vim](https://github.com/davidoc/taskpaper.vim) |タスク・TODOリストを管理
|[unite.vim](https://github.com/Shougo/unite.vim)     |ファイラとして利用
|[solarized](https://github.com/altercation/vim-colors-solarized)     |カラースキーム
|[surround](http://www.vim.org/scripts/script.php?script_id=1697)      |HTMLタグなど"囲まれているもの"の編集補助


