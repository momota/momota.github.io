---
layout: post
title: "Google Apps Script でサーバレスな為替レート取得クローラをつくる"
date: 2018-06-25 14:35
comments: true
categories: gas google javascript serverless
---
Google [Apps Script](https://script.google.com/home) (GAS) は、Google が提供する JavaScript プラットフォームで、Google apps (Calendar, Docs, Drive, Gmail, Sheets, and Slides) に対して処理する JavaScript を簡単に書ける。
Excel マクロのすごい版みたいな感覚。

このスクリプトからHTTP GETリクエストを出したり、受け付けたりできる。

今回は、この GAS を使って、無料の Web クローラを作ってみる。

<!-- more -->

## 処理の流れ

1. GAS から為替レート API をたたく
2. 取得した為替レートデータを Google Sheets へ出力する
3. 上記を1分間隔で実行する。


## 1. GAS から為替レート API をたたく

### 為替レート API の確認

API は https://www.gaitameonline.com/rateaj/getrate を利用する。
ここへ HTTP GET すると以下のような JSON フォーマットが返ってくる。 (整形済み)

```json
{
  quotes=[
    {
      high=1.9196,
      low=1.9162,
      ask=1.9212,
      bid=1.9195,
      currencyPairCode=GBPNZD,
      open=1.9167
    },
    {
      high=82.83,
      low=82.35,
      ask=82.41,
      bid=82.36,
      currencyPairCode=CADJPY,
      open=82.76
    },
    // ... (snip) ...
```

[robots.txt](https://www.gaitameonline.com/robots.txt) には特に言及されていないが、非常識なアクセスはしないように注意。


### GAS の作成

[Apps Script](https://script.google.com/home) のメニューから `+ 新規スクリプト` をクリックすると、新規プロジェクトが立ち上がる。
プロジェクト名やスクリプト名 (デフォルト `Code.gs`) は適当に変更する。


GAS で API をたたく処理は以下のように書ける。

`UrlFetchApp.fetch(url)` 部分で API をたたきにいっている。
`fx.date = now` で、取得した JSON に取得日時フィールド `date` を足している。

```javascript
function callExchangeAPI() {
    var now      = new Date(),
        url      = "https://www.gaitameonline.com/rateaj/getrate",
        response = UrlFetchApp.fetch(url),
        content  = response.getContentText(),
        fx       = JSON.parse(content);

    fx.date = now;
    Logger.log(fx);
    return fx;
}
```

上記をコピペして実行する (▶ボタンを押す) と初回実行時には以下のような確認ダイアログがでるので `Review Permissions` を押す。

![01_notice_initial_run](/images/20180625_gas/01_notice_initial_run.png)

その後、Choose an account画面で自分のアカウントを選ぶと、確認画面 `YOUR-PROJECT-NAME wants to access your Google Account` が出てくるので `ALLOW` ボタンを押す。そうすると実行できる。

`Logger.log(fx);` 部分でログ出力しているので、メニュー `View` > `Logs` からログ出力できていることが確認できる。

```
[18-06-24 23:24:06:486 PDT] {date=Mon Jun 25 15:24:06 GMT+09:00 2018, quotes=[{high=1.9224, low=1.9162, ask=1.9223, bid=1.9206, currencyPairCode=GBPNZD, open=1.9167}, {high=82.83, low=82.22, ask=82.45, bid=82.40, currencyPairCode=CADJPY, open=82.76}, {high=1.7870, low=1.7817, ask=1.7855, bid=1.7846, currencyPairCode=GBPAUD, open=1.7817}, 
... snip ...
```


## 2. 取得した為替レートデータを Google Sheets へ出力する

取得した為替レートを以下のように Google sheetsに出力する。

![03_sheet](/images/20180625_gas/03_sheet.png)

1行目はヘッダ、列の定義は以下。通貨ペア種別すべての Ask や Bid などを1行で表現する。

```
date, currencyPairCode1, high1, low1, ask1, bid1, open1, currencyPairCode2, high2, ...
```

まずは、Google sheetsを新規作成し、その URL をコピーする。

個別の URL をコードに埋め込みたくないので、`Script properties` に値をセットしてそれをコードの中から利用する。

GAS のメニュー `File` > `Project properties` から `Script properties` タブへ移動する。
`+ Add row` のリンクから行をエントリーする。
`Property` に SHEET_URL、`Value` に先程コピーした Google sheets の URLを登録する。
登録するURL は `https://docs.google.com/spreadsheets/d/xxxxxxxxxxxxxxxxxxxxxxxxxxx/edit` のような形式。


上述した `Script properties` から値を参照して、Sheet オブジェクトを取得する処理は以下。

```javascript
  function() {
    if(this.getSheet.sheet) { return this.getSheet.sheet; }

    var SHEET_URL = PropertiesService.getScriptProperties().getProperty('SHEET_URL');
    if (!SHEET_URL) {
      throw 'You should set "SHEET_URL" property from [File] > [Project properties] > [Script properties]';
    }

    var sheets = SpreadsheetApp.openByUrl(SHEET_URL);
    this.getSheet.sheet = sheets.getActiveSheet();
    return this.getSheet.sheet;
  }
```

以下のようにAPI から取得してきた JSON データをシートに出力する。

```javascript
  function(ex_json) {
    var sheet = this.getSheet();

    // データ追加行 (Sheetの最終行 + 1) を取得する
    var last_row = sheet.getLastRow() + 1;
  
    var col = 1;
    sheet.getRange(last_row, col++).setValue(ex_json.date);
  
    for each(var quote in ex_json.quotes) {
      sheet.getRange(last_row, col++).setValue(quote.currencyPairCode);
      sheet.getRange(last_row, col++).setValue(quote.high);
      sheet.getRange(last_row, col++).setValue(quote.low);
      sheet.getRange(last_row, col++).setValue(quote.ask);
      sheet.getRange(last_row, col++).setValue(quote.bid);
      sheet.getRange(last_row, col++).setValue(quote.open);
    }
  }
```

上記をまとめると以下。
コピペして実行する (▶ボタンを押す) と Google sheetsに為替データが挿入される。
初回実行時に権限の確認ダイアログがでると思うが許可してあげる。

```javascript
// Main
function scrapeExchangeToSheet() {
  var ex_json = exchange.callExchangeAPI();
  exchange.writeSheets(ex_json);
}

var exchange = {
  getSheet: function() {
    if(this.getSheet.sheet) { return this.getSheet.sheet; }

    var SHEET_URL = PropertiesService.getScriptProperties().getProperty('SHEET_URL');
    if (!SHEET_URL) {
      throw 'You should set "SHEET_URL" property from [File] > [Project properties] > [Script properties]';
    }

    var sheets = SpreadsheetApp.openByUrl(SHEET_URL);
    this.getSheet.sheet = sheets.getActiveSheet();
    return this.getSheet.sheet;
  },

  // call exchange API
  callExchangeAPI: function() {
    var now      = new Date(),
        url      = "https://www.gaitameonline.com/rateaj/getrate",
        response = UrlFetchApp.fetch(url),
        content  = response.getContentText(),
        fx       = JSON.parse(content);

    fx.date = now;
    return fx;
  },

  // Write exchange data (JSON) to the Google Sheet
  writeSheets: function(ex_json) {
    var sheet = this.getSheet();

    // get last row to add exchange data
    var last_row = sheet.getLastRow() + 1;
  
    var col = 1;
    sheet.getRange(last_row, col++).setValue(ex_json.date);
  
    for each(var quote in ex_json.quotes) {
      sheet.getRange(last_row, col++).setValue(quote.currencyPairCode);
      sheet.getRange(last_row, col++).setValue(quote.high);
      sheet.getRange(last_row, col++).setValue(quote.low);
      sheet.getRange(last_row, col++).setValue(quote.ask);
      sheet.getRange(last_row, col++).setValue(quote.bid);
      sheet.getRange(last_row, col++).setValue(quote.open);
    }
  }
}
```

## 3. 上記を1分間隔で実行する。

上記の処理を定期的に実行する。

時計マークのアイコンをクリックする。もしくは、メニュー `Edit` > `Current project's trigger` を選択する。
そうするとダイアログが出てくるので、`No triggers set up. Click here to add one now.` をクリックする。

`Run` にはメイン処理の `scrapeExchangeToSheet`, `Events` には好きな起動時間を設定できる。
ここでは1分間隔で実行するため、`Time-driven : Minutes timer: Every minute` を選択する。


1分間隔で Google sheets にデータが挿入されていることが確認できる。
