---
layout: post
title: "seleniumノウハウ"
date: 2016-05-28 16:57
comments: true
categories: selenium ruby
---


![selenium](/images/selenium.png)

[ブラウザ操作の自動化: Selenium と Ruby](http://momota.github.io/blog/2016/02/26/selenium/) でも書いたが、
selenium が便利すぎて、最近よくスクリプトを書くようになった。

以下のようなノウハウが溜まってきたので、ここらで放出する。

- ウィンドウサイズのリサイズ
- ウィンドウ位置の移動
- スクリーンショットの取得
- 要素セレクタメソッドの使い分け
- ドロップダウンリストの選択
- マウスオーバ (hover)
- フレーム移動
- ポップアップダイアログの扱い
- コード量を減らすためのモンキーパッチ
- よく採るプログラム構造



インストール方法などは [ブラウザ操作の自動化: Selenium と Ruby](http://momota.github.io/blog/2016/02/26/selenium/) 
を参照。

selenium用スクリプト開発のノウハウというべきかコツといういべきかアレな感じだが、結局、
スクレイピングと同様にHTML構造を理解することが必要なのでブラウザ付属の開発ツールが
手離せない。

- [Firefox 開発ツール](https://developer.mozilla.org/ja/docs/Tools)
- [chrome devtools](https://developer.chrome.com/devtools)

<!-- more -->

これ以降のサンプルコードは以下を定義しているものとする。
```ruby
require "selenium-webdriver"

driver = Selenium::WebDriver.for :firefox
```

ウィンドウサイズのリサイズ
--------------------------

`driver.manage.window.resize_to`の引数にリサイズするサイズ情報(width, height)を渡す。

```ruby
width  = 100
height = 100
driver.manage.window.resize_to(width, height)
```

現在のサイズを取得して、相対的にリサイズしたい場合は以下のようにする。

```ruby
size = driver.manage.window.size
size.width  += 100
size.height += 100
driver.manage.window.size = size
```


ウィンドウ位置の移動
--------------------

`driver.manage.window.move_to`の引数に移動したい場所の座標情報(x, y)を渡す。

```ruby
x = 100
y = 100
driver.manage.window.move_to(x, y)
```

現在のウィンドウ位置を取得して、相対的にリサイズしたい場合は以下のようにする。

```ruby
pos = driver.manage.window.position
pos.x += 100
pos.y += 100
driver.manage.window.position = pos
```


スクリーンショットの取得
------------------------

開いているページのスクリーンショットを撮りたい場合は、`driver.save_screenshot` に保存先のファイル名を指定するだけ。

```ruby
driver.get("https://www.google.co.jp/")
driver.save_screenshot("/path/to/save/screenshot.png")
```

ページ全体を撮りたいのに、frame 構造が邪魔をして、スクロールしなければ全体が撮れない場合がある。
ちょっと、調べたところ `driver.execute_script("some script")` で javascript を利用してスクロールをしている人がいたり、
`Selenium::WebDriver::Element.location_once_scrolled_into_view` を使ってスクロールしている人がいたり。
どちらにしても、コンテンツ表示上の高さを取得して、1スクロール分の長さをデクリメントしていっているような処理。

自分の場合は、めんどくさかったので、上述したリサイズ方法を使ってスクロールがいらないくらいウィンドウサイズを大きくして、
スクリーンショットを撮るという荒業を繰り広げている。


要素セレクタメソッドの使い分け
------------------------------

要素セレクタメソッドは `find_element` と `find_elements` の2種類。
そのメソッド名の単数形・複数形の通りなのだが、以下のような違いがある

- `find_element`
  - 指定した引数にマッチする最初の要素 (`WebDriver::Element`) を **1つ** 返す。
  - マッチする要素がなければ例外 (`NoSuchElementError`) を投げる。
- `find_elements`
  - 指定した引数にマッチする要素を詰めた配列 (`Array<WebDriver::Element>`) を返す。
  - マッチする要素がなければ、空の配列 (`Array`) を返す。


テーブルの `<tr>` 要素やリストの `<li>` 要素に対してイテレーション処理するときには `find_elements` が便利。
以下は、例。

```ruby
# リストのリンクをクリックしていくイテレーションの例
elements = driver.find_elements(:css, "#some-list > li")
elements.each { |e|
  link = e.find_element(:tag_name, "a")
  link.click
}
```


要素セレクタメソッドの引数には、`find_element(:how, "what")` のように symbol と文字列を渡す。
渡す引数は、`find_elements` も同じ。

指定できるsymbolの種類は以下。

symbol             | 対象
-------------------|---------------------------------
:class             | クラス名 (属性名 class)
:class_name        | 上記 :class と同じ
:id                | ID (属性名 id)
:link_text         | `<a>` タグのテキスト
:link              | 上記 :link_text と同じ
:partial_link_text | `<a>` タグのテキストの部分文字列
:name              | name (属性名 name)
:tag_name          | タグ名
:xpath             | xpath で指定
:css               | css セレクタ で指定


[参考 Module: Selenium::WebDriver::Find](http://www.rubydoc.info/gems/selenium-webdriver/0.0.28/Selenium/WebDriver/Find)


ドロップダウンリストの選択
--------------------------

Selenium IDEでRubyコードの出力をしようとすると、ドロップダウンリストの選択をする処理部分がERRORに
なってコメントアウトされることがある。(今のSelenium IDEバージョンでは大丈夫そう)

ドロップダウンリストの選択は以下のように書けばOK。

```ruby
s = Selenium::WebDriver::Support::Select.new(driver.find_element(:tag_name, "select"))
s.select_by(:text, "ほげほげ")  # 表示テキストで選択
s.select_by(:value, "value1")   # valueの値で選択
s.select_by(:index, 0)          # index(0, 1, 2, ...)で選択

```



マウスオーバ (hover)
--------------------

例えば、ナビゲーションメニューなどが、通常時には折りたたまれていて、メニュー上にマウスオーバした場合に、
子メニューが展開されるようなページがある。
折りたたまれているときに、子メニューHTMLをロードできていないようなページのときは、ユーザ操作と同じように
マウスオーバしてあげる必要がある。

以下のようにマウスオーバしたい要素を指定して、`driver.mouse.move_to` を呼べば良い。

```ruby
e = driver.find_element(:id, "menu")
driver.mouse.move_to( e )
```


フレーム移動
------------

frame や iframe 要素を使っているサイトで、そのフレーム内の要素に対して操作したい場合、当該フレームへ切り替える操作が必要となる。

以下のようにスイッチしたいフレーム要素を指定して、`driver.switch_to.frame` を呼べば良い。

```ruby
frame = driver.find_element(:id, "frame")
driver.switch_to.frame( frame )
```

フレームを移動してから、いったん最上位のフレームにに戻りたい場合は、以下。

```ruby
driver.switch_to.window( driver.window_handle )

frame = driver.find_element(:id, "top-frame")
driver.switch_to.frame( frame )
```



ポップアップダイアログの扱い
----------------------------

「マジでこのページから移動する?」的なjavascriptによるポップアップダイアログを表示するページがある。
このポップアップダイアログを強制的に閉じるには以下のようにする。

`.accept` ではい、`.dismiss`でいいえを押して閉じる。

```ruby
begin
  alert_box = driver.switch_to.alert
  alert_box.accept     # 「はい」「OK」ボタンクリック
  # alert_box.dismiss  # 「いいえ」「cancel」ボタンクリック
rescue Selenium::WebDriver::Error::NoSuchAlertError => e
  # ダイアログがポップアップしない場合、例外を投げるので、何もせずにスルー
end
```



コード量を減らすためのモンキーパッチ
------------------------------------

たとえば、ログインなどの処理の際、フォームへ文字列を送る `send_keys` 前にいつも `.clear` しているので
もう `send_keys` の中に `.clear` 処理を入れ込んでしまえと思った。

また、`input` タグの値を取得するときは、`.attribute("value")` と長ったらしく書く必要があるので `.value`メソッドを
定義してしまおうと思った。checkbox のチェック有無も同じく。

そこで、以下のようなモンキーパッチを書く。

```ruby
module ElementExtension
  refine Selenium::WebDriver::Element do
    def send_keys( str )
      self.clear
      super( str )
    end

    def value
      self.attribute("value")
    end

    def checked?
      self.attribute("checked").nil? ? false : true
    end
  end
end

using ElementExtension
```

そうすると以下のようにコード量を減らせる。

```diff
# たとえば、ログイン処理
user     = "USER"
password = "PASSWORD"

- driver.find_element(:id, "loginuser").clear
driver.find_element(:id, "loginuser").send_keys( user )
- driver.find_element(:id, "loginpass").clear
driver.find_element(:id, "loginpass").send_keys( password )
driver.find_element(:id, "submit").click


# <input>の値を取得する処理
- hostname = driver.find_element(:id, "hostname").attribute("value")
+ hostname = driver.find_element(:id, "hostname").value

# チェックボックスがチェックされているかどうか
checkbox = driver.find_element(:id, "some-checkbox")
- do_something if checkbox.attribute("checked").nil?
+ do_something if checkbox.checked?
```


モンキーパッチの書き方は以下を参考にした。

- [分別のあるRubyモンキーパッチャーになるために](http://melborne.github.io/2013/08/30/monkey-patching-for-prudent-rubyists/)



プログラム構造
--------------

よく採るプログラム構造は以下。

- 操作ターゲットとなるサイト単位にクラスを作る
- 処理の内容にしたがって、メソッドを作る。以下の様な粒度。
  - サインイン
  - リスト表示
  - 自動入力と申請
- アカウント情報や入力データはYAMLファイルなどに書き出しておく


```ruby
# conding: utf-8
require "selenium-webdriver"
# その他必要に応じてrequire


module Elementextension
  # 上記モンキーパッチの章を参照
end
using Elementextension


class SomeSite
  def initialize
    # 初期化処理
    # 定数をセットしたり、webdriverを生成してアクセスしたり。
    @ss_dir  = "./screenshot/"
    FileUtils.mkdir_p( @ss_dir ) 

    @driver  = Selenium::WebDriver.for :firefox
    base_url = "http://somesite.com"
    @driver.get(base_url + "/index.html")
  end

  def sign_in(user, password)
    # サインイン処理
  end

  def do_something
    # やりたいことを書く
  end

  def close
    # 終了時に実行する処理を書く
  end

  private

  def get_screenshot( str )
    now = DateTime.now.strftime("%Y%m%d%H%M%S")
    @driver.save_screenshot(@ss_dir + "_" + str + "_" + now + ".png")
  end
end


# ----------------------------------------------------------------------
# main
# ----------------------------------------------------------------------
if __FILE__ == $0
  # サインインに必要なアカウント情報はYAMLなどに書いておいて、それを読む
  y        = YAML.load_file("/path/to/yaml.yml")
  user     =  y["account"]["user"]
  password =  y["account"]["password"]

  hoge = SomeSite.new
  hoge.sign_in(user, password)
  hoge.do_something
  hoge.close
end
```



関連
----

- [ブラウザ操作の自動化: Selenium と Ruby](http://momota.github.io/blog/2016/02/26/selenium/) 

