---
layout: post
title: "Flask アプリ (チャットボット) のコンテナ化"
date: 2018-10-11 10:54
comments: true
categories: bot python docker container
---
以下で作ったようなFlaskアプリ (チャットボット) をコンテナ化 (Docker) する。

- [Python unittest で Flask (チャットボット) の単体テスト - momota.txt](http://momota.github.io/blog/2018/06/19/unittest/)

<!-- more -->


## Flask アプリ

Python のバージョンは 3.6。

Flask アプリのディレクトリ構成は以下。

```
$ tree
.
|____log
| |____mattermost_bot.log
|____mattermostbot.py
```

- `mattermostbot.py`: ボット本体
- `log/` : ログ保存用のディレクトリ

`mattermost.py` は簡易的なオウム返しするボットで以下のようなスクリプト。

```python
import json
import logging
from flask import Flask, request
from logging.handlers import RotatingFileHandler

SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
BOT_LOG    = SCRIPT_DIR + '/log/mattermost_bot.log'

app = Flask(__name__)

@app.route('/bot', methods=['POST'])
def bot():
    # mattermost -> bot へ送信される JSON データの取得
    post_dict = request.form
    app.logger.info(post_dict)

    # JSONから token と text (ユーザが入力したメッセージ) を取得
    token = post_dict['token']
    income_text = post_dict['text']

    # income_text は `bot_name COMMAND ARGUMENT` のような形式なので
    # 半角スペースで分割し、それぞれの要素を変数に格納する
    text_array = income_text.split(' ')
    bot_name = text_array[0]
    command = text_array[1]
    arg = " ".join(text_array[2:])

    payload_text = ""

    # command によって処理を分岐する
    if command == "echo":
        payload_text = echo(arg)

    app.logger.info(payload_text)

    # レスポンス用の JSON を組み立てる
    payload = {
        'username': bot_name,
        'icon_url': 'http://your-server/images/bot_icon.png',
        'text': payload_text,
        'MATTERMOST_TOKEN': token
    }
    json_payload = json.dumps(payload)

    return json_payload

# ログ出力用メソッド
def log(app):
    handler = RotatingFileHandler(BOT_LOG, maxBytes=10000, backupCount=2)
    handler.setLevel(logging.INFO)
    formatter = logging.Formatter('%(asctime)s\t%(lineno)d\t%(levelname)s\t%(name)s\t%(message)s')
    handler.setFormatter(formatter)
    app.logger.addHandler(handler)

# -------------------------------------------------------
# echo command
# -------------------------------------------------------
def echo(text):
    return text


if __name__ == '__main__':
    log(app)
    app.debug = True
    app.run(host='0.0.0.0')
```

## Requirements File

コンテナにインストールする必要のある Python パッケージ を `requirements.txt` に書き出す。ようは `pip` でインストールやつを列挙する。

```
Flask==0.12.2
```

手で作らなくても、すでに Flask アプリを動かしている環境があれば、以下のコマンドで出力できる。

```sh
$ pip freeze > requirements.txt
```

## Dockerfile

Docker イメージを生成するため、以下のような `Dockerfile` を作成する。

```
FROM python:3.6

ARG project_dir=/chatbot/

ADD requirements.txt $project_dir
ADD mattermostbot.py $project_dir
ADD log "${project_dir}log"

WORKDIR $project_dir

RUN pip install -r requirements.txt
CMD ["python", "mattermostbot.py"]
```

プロキシ環境下でビルドする場合は、以下のようにコンテナ内の環境変数を設定してあげれば良い。
追記する場所は、`RUN pip install -r requirements.txt` より前。

```
ARG proxy_host="proxy.co.jp"
ARG proxy_port="85"
ARG proxy_user="USER"
ARG proxy_pass="PASSWORD"
ARG PROXY="${proxy_user}:${proxy_pass}@${proxy_host}:${proxy_port}"

ENV http_proxy="http://${PROXY}" \
    https_proxy="https://${PROXY}" \
    ftp_proxy="ftp://${PROXY}"
```

## Docker イメージのビルド

以下のコマンドで Docker イメージをビルドする。

```sh
$ docker build -t mattemost_bot:latest .
```

できたイメージは以下で確認できる。

```sh
$ docker images
```

## コンテナの実行

```sh
$ docker run -p 5000:5000 -it mattermost_bot
```

起動状態などの詳細は以下で確認できる。

```sh
$ docker ps -a
```
