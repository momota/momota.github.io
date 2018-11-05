---
layout: post
title: "serverless framework による AWS Lambda ローカル開発"
date: 2018-11-05 20:11
comments: true
categories: serverless serverless-framework faas lambda
---

![serverless framework logo](/images/20181105_serverless-framework/serverless-framework-logo.png)

[Serverless Framework](https://serverless.com/) を使うことにより、FaaS (AWS Lambda, GCP Cloud Functions, Azure functions, など) の開発をローカルで実施できる。
ローカル環境で自分の好きなエディタ・IDEで開発やテストが可能になるし、デプロイも容易になる。

AWS には AWS SAM もあるが、他のクラウドプロバイダでも開発物やノウハウが使い回せることが期待できるので、3rd パーティ製の Serverless Framework を選ぶ。

本稿では、Serverless Framework の導入と、Hello Worldアプリ (AWS Lambda) のデプロイについて書く。


## 環境

```sh
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=18.04
DISTRIB_CODENAME=bionic
DISTRIB_DESCRIPTION="Ubuntu 18.04.3 LTS"
```


<!-- more -->


## Serverless framework のインストール


まず Node.js (v6.5.0以降) と npm をインストールする。

```sh
$ sudo apt-get update

$ sudo apt-get install nodejs
$ nodejs --version
v8,10,0

$ sudo apt-get install npm
$ npm --version
3.5.2
```

npm で serverless framework をインストールする。

```sh
$ sudo npm install -g serverless
$ serverless --version
1.32.0
```


## AWS アクセスキーを設定する

1. AWSアカウントを作り、マネージドコンソールからIAMページにアクセスする
2. ペインのユーザをクリックし、「ユーザを追加」をクリックする
    - 適切なユーザ名を入力する
    - アクセスの種類の、「プログラムによるアクセス」にチェックをいれ、次へ
    - アクセス許可の設定は、「既存のポリシーを直接アタッチ」で、「AdministratorAccess」を選択し、次へ
    - レビューして、問題なければ作成する
3. アクセスキーIDとシークレットアクセスキーをコピーする
4. Serverless framework に設定する
```sh
# パターン1: 環境変数に設定
$ export AWS_ACCESS_KEY_ID=<your-key-here>
$ export AWS_SECRET_ACCESS_KEY=<your-secret-key-here>

# パターン2: serverless config credentials コマンドで設定
$ serverless config credentials --provider aws --key <your-key-here> --secret <your-secret-key-here>
# => ~/.aws/credentials が生成される
```


## Serverless framework で Hello World

まず、作業ディレクトリの作成と、`package.json` を作成する。

```sh
$ mkdir my-express-application && cd my-express-application
$ npm init -f
```

いくつかのライブラリをインストールする。

```sh
$ npm install --save express serverless-http
```

アプリケーションを `index.js` に書く。

```javascript
const serverless = require('serverless-http');
const express = require('express');
const app = express();

app.get('/', function (req, res) {
  res.send(JSON.stringify({
    id: '00001',
    message: 'Hello World!',
  }));
});

module.exports.handler = serverless(app);
```

これはルートパス `/` にアクセスがあった場合に、 `{ id: '00001', message: 'Hello World!' }` を返す単純なアプリ。

これをデプロイするため、以下の `serverless.yml` を作成する。

```yaml
service: my-express-application

provider:
  name: aws
  runtime: nodejs8.10
  stage: dev
  region: ap-northeast-1

functions:
  app:
  handler: index.handler
  events:
    - http: ANY /
    - http: 'ANY {proxy+}'
```

関数をデプロイする。

```sh
$ serverless deploy
...snip...
Service Information
service: my-express-application
stage: dev
region: ap-northeast-1
stack: my-express-application-dev
api keys:
  None
endpoints:
  ANY - https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/dev
  ANY - https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/dev/{proxy+}
functions:
  app: my-express-application-dev-app
```

数分後にデプロイが完了し、`endpoints` の情報が出力される。
この URL へアクセスし、動作確認する。

```sh
$ curl -X GET https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/dev
{"id":"00001","message":"Hello World!"}
```

アプリケーションで定義した JSON が返ってくる。

## 参考

- [Serverless Framework - AWS Lambda Guide - Quick Start](https://serverless.com/framework/docs/providers/aws/guide/quick-start/)
- [Serverless Framework - AWS Lambda Guide - Credentials](https://serverless.com/framework/docs/providers/aws/guide/credentials/)
- [Deploy a REST API using Serverless, Express and Node.js](https://serverless.com/blog/serverless-express-rest-api/)
