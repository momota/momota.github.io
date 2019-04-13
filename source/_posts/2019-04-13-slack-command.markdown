---
layout: post
title: "ねこの画像をランダムに表示する slack command"
date: 2019-04-13 19:06
comments: true
categories: slack google gcp functions serverless faas serverless-framework chatops python
---

心の平穏のために、猫の画像をランダムに表示してくれる Slack command `/neko` を作った。

Slack で `/neko` と打つと、Cloud Functions にリクエストが飛び、Functions が [TheCatAPI](https://thecatapi.com/) からランダムに猫画像 URL を取得するもの。

![slack neko command](/images/20190413_neko-slack-command/neko-command.png)


- 参考
  - [serverless framework による AWS Lambda ローカル開発 - momota.txt](http://momota.github.io/blog/2018/11/05/serverless-framework/)
  - [Google cloud functions で slack (slash) commands をつくる - momota.txt](http://momota.github.io/blog/2019/04/06/slack-commands-by-cloud-functions/)

 使ったもの                  | 用途                
-----------------------------|---------------------
 Google Cloud Functions      | バックエンド        
 Python 3.7                  | Functionsの実装言語 
 Serverless framework 1.40.0 | GCPへのデプロイ     

<!-- more -->


## Serverless framework の初期設定

```sh
# create new service
$ serverless create --template google-python --path neko

# go to the service directory
$ cd neko

# install provider plugins
$ npm install
```

`serverless.yml` を自分の GCP 環境に合わせて編集する。

```yaml
service: neko

provider:
  name: google
  stage: dev
  runtime: python37
  region: YOUR-REGION
  project: YOUR-GCP-PROJECT-ID
  credentials: ~/.gcloud/YOUR-KEYFILE.json

plugins:
  - serverless-google-cloudfunctions

package:
  exclude:
    - node_modules/**
    - .gitignore
    - .git/**

functions:
  neko-command:
    handler: neko
    events:
      - http: path
    memorySize: 256
    timeout: 60s
    labels: {
        application: slack-slash-command,
        environment: development,
        owner: momota
}
```


## neko コマンドの実装

まず [TheCatAPI](https://thecatapi.com/) の API エンドポイントをコードの中に埋め込みたくないので、 `config.json` という外部ファイルに書き出し、実行時にそこから読み込むことにする。

以下のような感じ。

```json
{
    "API_URL": "https://api.thecatapi.com/v1/images/search?format=json"
}
```

Functions 用のコードは以下のような感じ。

```python
import json
import requests
from flask import jsonify

# 外部ファイル config.json の読み込み
with open('config.json', 'r') as f:
    data = f.read()
config = json.loads(data)

# TheCatAPI を叩いて、レスポンス (JSON) から 猫画像 URL を取得
def neko_url():
    res = requests.get(config['API_URL'])
    json_res = json.loads(res.text)
    return json_res[0]['url']

# Slackへの応答フォーマットに整形
def format_slack_message(message):
    return {
        'response_type': 'in_channel',
        'text': message
    }

# Handler: Functions がリクエストを受けたときに実行する関数
def neko(request):
    message = neko_url()
    response = format_slack_message('ねこです ' + message)
    return jsonify(response)
```

あとは GCP にデプロイして、Slack の Slash command の設定をすれば使える。

```sh
$ serverless deploy -v
```

## ちょっと頑張る： Slack トークン認証とエラーハンドリング

Slack の Verification Token による簡易的な認証処理を追加する。

Slack command のApp設定から、Basic Information > App Credentials > Verification Token からトークンを取得して、`config.json` に書く。

```json
{
    "API_URL": "https://api.thecatapi.com/v1/images/search?format=json",
    "SLACK_TOKEN": "YOUR-VERIFICATION-TOKEN"
}
```

以下のような処理を追加し、認証処理とエラーハンドリング処理を追加する。

とりあえず意図しないリクエストがバックエンド (Functions) にきたら、例外を投げる。


```python
def verify_request(request):
    # POSTメソッドじゃない
    if request.method != 'POST':
        raise Exception(405, 'Only POST requests are accepted')

    # データが POST されていない
    if not request.form:
        raise Exception(422, 'POST form is empty')

    # POST データに ’token’ フィールドにない
    if 'token' not in request.form:
        raise Exception(422, 'POST form is invalid')

    # トークンがマッチしない
    if request.form['token'] != config['SLACK_TOKEN']:
        raise Exception(403, 'Permission denied')
```

メイン処理側では、例外を補足したら、それに応じたエラーを返すように変更する。

```python
def neko(request):
    try:
        # validate request
        verify_request(request)

        message = neko_url()
        response = format_slack_message('ねこです ' + message)
        return jsonify(response)
    except Exception as e:
        code, msg = e.args
        print('Exception occured: <{}> {}'.format(code, msg))
        return (msg, code)
```

## もうちょっと頑張る: 非同期処理

[前回](http://momota.github.io/blog/2019/04/06/slack-commands-by-cloud-functions/)も述べたが、
Slack command は 3000 ms (3秒) 以内に応答しないとタイムアウトになってしまう。

[TheCatAPI](https://thecatapi.com/) のような外部 API に依存している場合、3 秒以内のレスポンスを保証できない可能性が高くなる。
そこで、バックエンドの Functions がリクエストを受けたら即時にレスポンスを返し、スレッドにより非同期で応答する。
Slack としても、そのような仕組みを支援するためにレスポンス用 URL を付与して、バックエンド側にリクエストを投げてくれる。


スレッド処理のため、メイン処理を関数化し、レスポンス用 URL へ返答するよう書き換える。

```python
from threading import Thread

def neko_async(response_url):
    message = neko_url()
    response = format_slack_message('ねこです ' + message)
    post_data = json.dumps(response)

    post_response = post_to_slack(response_url, post_data)
    return

def neko(request):
    try:
        # validate request
        verify_request(request)

        # レスポンス用 URL の取得
        response_url = request.form['response_url']

        # スレッドで猫画像 URL処理を取得
        thread = Thread(target=neko_async, kwargs={'response_url': response_url})
        thread.run()

        return ''
    except Exception as e:
        code, msg = e.args
        print('Exception occured: <{}> {}'.format(code, msg))
        return (msg, code)
```

最終的には以下のような感じになる。

```python
import json
import requests
from threading import Thread

# 外部ファイル config.json の読み込み
with open('config.json', 'r') as f:
    data = f.read()
config = json.loads(data)

# エラーハンドリングと簡易認証処理
def verify_request(request):
    # POSTメソッドじゃない
    if request.method != 'POST':
        raise Exception(405, 'Only POST requests are accepted')

    # データが POST されていない
    if not request.form:
        raise Exception(422, 'POST form is empty')

    # POST データに ’token’ フィールドにない
    if 'token' not in request.form:
        raise Exception(422, 'POST form is invalid')

    # トークンがマッチしない
    if request.form['token'] != config['SLACK_TOKEN']:
        raise Exception(403, 'Permission denied')

# スレッド処理用関数
def neko_async(response_url):
    message = neko_url()
    response = format_slack_message('ねこです ' + message)
    post_data = json.dumps(response)

    post_response = post_to_slack(response_url, post_data)
    return

# TheCatAPI を叩いて、レスポンス (JSON) から 猫画像 URL を取得
def neko_url():
    res = requests.get(config['API_URL'])
    json_res = json.loads(res.text)
    return json_res[0]['url']

# Slackへの応答フォーマットに整形
def format_slack_message(message):
    return {
        'response_type': 'in_channel',
        'text': message
    }

# Slack レスポンス用 URL への POST処理
def post_to_slack(url, post_data):
    post_headers = {
        'Content-type': 'application/json; charset=utf-8'
    }

    return requests.post(
        url,
        data=post_data,
        headers=post_headers
    )

# Handler: Functions がリクエストを受けたときに実行する関数
def neko(request):
    try:
        # validate request
        verify_request(request)

        response_url = request.form['response_url']
        thread = Thread(target=neko_async, kwargs={'response_url': response_url})
        thread.run()

        return ''
    except Exception as e:
        code, msg = e.args
        print('Exception occured: <{}> {}'.format(code, msg))
        return (msg, code)
```
