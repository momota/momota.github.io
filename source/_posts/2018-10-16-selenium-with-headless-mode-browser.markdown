---
layout: post
title: "Windows + Chrome Headless mode + Ruby + Selenium"
date: 2018-10-16 11:31
comments: true
categories: windows ruby selenium chrome headless
---

Selenium と Ruby によりテストの自動化であったり、入力の自動化を進めている。
コマンドラインから実行すればよいことがほとんどで、ブラウザのGUIが起動しなくてもよい。
したがって、GUI が起動しない Google Chrome の Headless モードを使った Selenium スクリプトへ順次移行していこうと思う。

動作の軽量化もちょっとは期待できるかも。 (比較計測はしていない)

以下は、関連記事。

- [ブラウザ操作の自動化： selenium と ruby - momota.txt](http://momota.github.io/blog/2016/02/26/selenium/)
- [seleniumノウハウ - momota.txt](http://momota.github.io/blog/2016/05/28/selenium-know-how/)


<!-- more -->


## 環境情報

ソフトウェア           | バージョン
-----------------------|----------------------------
OS                     | Windows 10 Pro 1709
ブラウザ               | Google Chrome 69.0.3497.100
ChromeDriver           | 2.39.562718
ruby                   | 2.5.1p57
gem selenium-webdriver | (3.14.1, 3.12.0)

Ruby と gem はインストールされているものとする。


## Chrome headless モードの動作確認

Chrome のバージョン59以降からヘッドレスモードを搭載している。
Chrome をインストールしていない場合は、インストールする。

動作確認はコマンドライン (Powershell or コマンドプロンプト) から可能で、以下のように動けば OK。

```
> cd "C:\Program Files (x86)\Google\Chrome\Application"
> ls


    ディレクトリ: C:\Program Files (x86)\Google\Chrome\Application


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----       2018/09/21     19:12                69.0.3497.100
d-----       2018/09/25     16:45                SetupMetrics
-a----       2018/09/15     17:47        1378648 chrome.exe
-a----       2018/09/25     16:45            410 chrome.VisualElementsManifest.xml
-a----       2018/05/17     17:06         122574 master_preferences

> .\chrome.exe --enable-logging --headless --disable-gpu --dump-dom https://www.chromestatus.com
<!DOCTYPE html>
<html lang="en"><head>
<meta charset="utf-8">
<title>Chrome Platform Status</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0">

<link rel="manifest" href="/static/manifest.json">
... (snip) ...
```

Proxy 環境下にいる場合は、`--proxy-server` や `--proxy-auth` オプションで指定してあげればよい。

```
> .\chrome.exe --enable-logging --headless --disable-gpu --dump-dom --proxy-server=http://PROXY_HOST:PORT --proxy-auth=USER:PASSWORD https://www.chromestatus.com
```


詳細は [ヘッドレス Chrome ことはじめ | Web | Google Developers](https://developers.google.com/web/updates/2017/04/headless-chrome?hl=ja) のあたりを参照。


## ChromeDriver

Selenium WebDriver は Selenium から Web ブラウザをコントロールするライブラリ群。
各ブラウザ向けに実装されていて、ChromeDriver は Chrome 用の WebDriver。

以下から入手し、PATHを通す。(もしくは通っているところに置く)

- [Downloads - ChromeDriver - WebDriver for Chrome](http://chromedriver.chromium.org/downloads)

コンソールからバージョンが表示できれば PATH 設定の確認になる。

```
> chromedriver.exe -v
ChromeDriver 2.39.562718 (9a2698cba08cf5a471a29d30c8b3e12becabb0e9)
```


## Selenium スクリプト

サンプルとして、本ブログのいくつかのページのスクリーンショットを取得する。
http://momota.github.io/blog/page/2/ ～ http://momota.github.io/blog/page/4/ のスクリーンショットを取得してみる。

Headless モードといっても特段難しいことはなく、以下のように Webdriver にオプションしてあげるだけでよい。

```ruby
options = Selenium::WebDriver::Chrome::Options.new
options.add_argument('--headless')
options.add_argument('--disable-gpu')
options.add_argument('--window-size=1929,2160')

driver = Selenium::WebDriver.for :chrome, options: options
```

Proxy の設定が必要な場合は、以下のようにオプションで指定する。
環境変数 `ENV['http_proxy']` に設定してもだめなので注意。

```diff
+ proxy_host = 'PROXY_HOST'
+ proxy_port = 'PORT'

options = Selenium::WebDriver::Chrome::Options.new
options.add_argument('--headless')
options.add_argument('--disable-gpu')
options.add_argument('--window-size=1929,2160')
+ options.add_argument("--proxy-server=http://#{proxy_host}:#{proxy_port}")
```

スクリプトとしては以下のような感じ。
普通に `ruby script.rb` で実行すると、スクリーンショット `実行ディレクトリ/screeshot_i.png` が出力される。


```ruby
# conding: utf-8
require 'selenium-webdriver'
require 'fileutils'

class Blog
  SCREENSHOT_DIR = './screenshot/'

  def initialize
    proxy_host = 'PROXY_HOST'
    proxy_port = 'PORT'

    options = Selenium::WebDriver::Chrome::Options.new
    options.add_argument('--headless')
    options.add_argument('--disable-gpu')
    options.add_argument('--window-size=1929,2160')
    options.add_argument("--proxy-server=http://#{proxy_host}:#{proxy_port}")

    @driver = Selenium::WebDriver.for :chrome, options: options
    @base_url = 'http://momota.github.io/'
    @accept_next_alert = true
    @driver.manage.timeouts.implicit_wait = 20
    @verification_errors = []

    FileUtils.mkdir(SCREENSHOT_DIR) unless Dir.exist?(SCREENSHOT_DIR)
  end

  def get_and_screenshot
    (2..4).each do |i|
      @driver.get("#{@base_url}blog/page/#{i}/")
      @driver.save_screenshot("#{SCREENSHOT_DIR}headles_#{i}.png")
      puts "access and save screenshot: #{i}"
    end
  end

  def close
    @driver.quit
  end
end

# ----------------------------------------------------------------------
# main
# ----------------------------------------------------------------------
if __FILE__ == $PROGRAM_NAME
  blog = Blog.new
  blog.get_and_screenshot
end
```
