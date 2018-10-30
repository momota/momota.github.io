---
layout: post
title: "kong でアクセス制御"
date: 2018-10-29 14:02
comments: true
categories: kong api restful acl security
---

![kong diagram](/images/20181029_acl-on-kong/kong-diag.png)

Kong を利用して、各 API に対して上記のようなアクセスを制御する。

- Consumer `aaa` は API a に対してアクセスできる。しかし、API b にはアクセスできない。
- Consumer `bbb` は API b に対してアクセスできる。しかし、API a にはアクセスできない。

Kong のプラグイン `key-auth` と `acl` により実装する。

各設定をまとめると以下。

ユーザ | service | route (kong path) | api key   | consumer | group
-------|---------|-------------------|-----------|----------|----------
aaa    | aaa-api | /aaa              | aaaaaaaaa | aaa      | aaa-group
bbb    | bbb-api | /bbb              | bbbbbbbbb | bbb      | bbb-group


<!-- more -->

## Service の設定

API a の service を定義する。

Service 名は `aaa-api`, 実体は mockbin.org。

```sh
$ curl -i -X POST --url http://localhost:8001/services \
> --data "name=aaa-api" \
> --data "url=http://mockbin.org"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 06:47:50 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 251

{
  "host": "mockbin.org",
  "created_at": 1540536470,
  "connect_timeout": 60000,
  "id": "b371024b-cebf-4065-a1bf-58ed9aad8dce",
  "protocol": "http",
  "name": "aaa-api",
  "read_timeout": 60000,
  "port": 80,
  "path": null,
  "updated_at": 1540536470,
  "retries": 5,
  "write_timeout": 60000
}
```


API b の service を定義する。

Service 名は `bbb-api`, 実体は www.gaitameonline.com。

```sh
$ curl -i -X POST http://localhost:8001/services/ \
> --data "name=bbb-api" \
> --data "url=https://www.gaitameonline.com/rateaj/getrate"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:08:42 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 278

{
  "host": "www.gaitameonline.com",
  "created_at": 1540537722,
  "connect_timeout": 60000,
  "id": "1d85e93c-8bc7-46f8-afed-a864ea8e0340",
  "protocol": "https",
  "name": "bbb-api",
  "read_timeout": 60000,
  "port": 443,
  "path": "/rateaj/getrate",
  "updated_at": 1540537722,
  "retries": 5,
  "write_timeout": 60000
}
```

## Route の定義

API a に route を設定する。

Consumer `aaa` には、Kong の `/aaa` という PATH を公開したいので、`paths[]=/aaa` を定義する。

```sh
$ curl -i -X POST http://localhost:8001/services/aaa-api/routes \
> --data "paths[]=/aaa"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 06:55:40 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 284

{
  "created_at": 1540536940,
  "strip_path": true,
  "hosts": null,
  "preserve_host": false,
  "regex_priority": 0,
  "updated_at": 1540536940,
  "paths": [
    "/aaa"
  ],
  "service": {
    "id": "b371024b-cebf-4065-a1bf-58ed9aad8dce"
  },
  "methods": null,
  "protocols": [
    "http",
    "https"
  ],
  "id": "8ecba70c-41c8-4152-b2c1-5caaca7e42ac"
}
```


API b にも同様に route を設定する。
Consumer `bbb` に、Kong の `/bbb` という PATHを公開する。

```sh
$ curl -i -X POST http://localhost:8001/services/bbb-api/routes \
> --data "paths[]=/bbb"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:12:23 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 284

{
  "created_at": 1540537943,
  "strip_path": true,
  "hosts": null,
  "preserve_host": false,
  "regex_priority": 0,
  "updated_at": 1540537943,
  "paths": [
    "/bbb"
  ],
  "service": {
    "id": "1d85e93c-8bc7-46f8-afed-a864ea8e0340"
  },
  "methods": null,
  "protocols": [
    "http",
    "https"
  ],
  "id": "6bea20ce-39c3-48ad-a912-948b33764453"
}
```

ここまでで、以下のようなプロキシ設定ができた

アクセス先                | プロキシ先
--------------------------|---------------------------------------------
http://localhost:8000/aaa | http://mockbin.org
http://localhost:8000/bbb | https://www.gaitameonline.com/rateaj/getrate


## key-auth プラグインによる API キー認証設定

key-auth プラグインを有効化し、API キーがない場合はアクセス拒否する。

各 service に対して有効化する。

```sh
$ curl -i -X POST http://localhost:8001/services/aaa-api/plugins --data "name=key-auth"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:15:31 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 276

{
  "created_at": 1540538131449,
  "config": {
    "key_in_body": false,
    "run_on_preflight": true,
    "anonymous": "",
    "hide_credentials": false,
    "key_names": [
      "apikey"
    ]
  },
  "id": "d54ca015-295b-4c5a-a8bf-718f05a80d3c",
  "service_id": "b371024b-cebf-4065-a1bf-58ed9aad8dce",
  "name": "key-auth",
  "enabled": true
}


$ curl -i -X POST http://localhost:8001/services/bbb-api/plugins --data "name=key-auth"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:16:19 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 276

{
  "created_at": 1540538179280,
  "config": {
    "key_in_body": false,
    "run_on_preflight": true,
    "anonymous": "",
    "hide_credentials": false,
    "key_names": [
      "apikey"
    ]
  },
  "id": "dd6fbb6c-29dd-47ad-b4b1-ac5f82065b20",
  "service_id": "1d85e93c-8bc7-46f8-afed-a864ea8e0340",
  "name": "key-auth",
  "enabled": true
}
```

API キーがないとアクセスできないことを検証する。

```sh
$ curl -i -X GET http://localhost:8000/aaa
HTTP/1.1 401 Unauthorized
Date: Fri, 26 Oct 2018 07:16:35 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
WWW-Authenticate: Key realm="kong"
Server: kong/0.14.1
Content-Length: 41

{
  "message": "No API key found in request"
}

$ curl -i -X GET http://localhost:8000/bbb
HTTP/1.1 401 Unauthorized
Date: Fri, 26 Oct 2018 07:17:05 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
WWW-Authenticate: Key realm="kong"
Server: kong/0.14.1
Content-Length: 41

{
  "message": "No API key found in request"
}
```

## Consumer の定義

Consumer `aaa` と `bbb` を作成する。

```sh
$ curl -i -X POST http://localhost:8001/consumers \
> --data "username=aaa"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:18:35 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 104

{
  "custom_id": null,
  "created_at": 1540432512,
  "username": "aaa",
  "id": "3cb5f512-ad07-4f27-9aed-4ddff00a020e"
}

$ curl -i -X POST http://localhost:8001/consumers \
> --data "username=bbb"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:18:40 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 104

{
  "custom_id": null,
  "created_at": 1540538320,
  "username": "bbb",
  "id": "9f8e8674-8d0d-43c9-8c58-59331949e0bd"
}
```

各 Consumer に API キーを発行する。

```sh
$ curl -i -X POST http://localhost:8001/consumers/aaa/key-auth --data "key=aaaaaaaaa"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:22:12 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 144

{
  "id": "c3c822ba-b676-48fe-b0ec-cf2507ec37d6",
  "created_at": 1540538532052,
  "key": "aaaaaaaaa",
  "consumer_id": "3cb5f512-ad07-4f27-9aed-4ddff00a020e"
}


$ curl -i -X POST http://localhost:8001/consumers/bbb/key-auth --data "key=bbbbbbbbb"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:22:30 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 144

{
  "id": "2102cddb-8b79-4c6d-bd77-a942430a0bb4",
  "created_at": 1540538550275,
  "key": "bbbbbbbbb",
  "consumer_id": "9f8e8674-8d0d-43c9-8c58-59331949e0bd"
}
```

以上の設定で、API キーがあれば API a, API b にはアクセスできるようになった。
しかし、Consumer `aaa` の API キーで、API a および API b の双方にアクセスできる。
Consumer `bbb` も双方の API にアクセスできる。

アクセス制御するために、ACL プラグインを利用する。


## acl プラグインの有効化

acl プラグインを利用することで、service や route 単位のアクセス制御が可能になる。
Consumer をグループに所属させ、グループ単位にアクセス許可する (`config.whitelist` に許可するグループを定義する) か、アクセス拒否する(`config.blacklist` に拒否するグループを定義する)。

ここでは、consumer `aaa` をグループ `aaa-group` へ、consumer `bbb` を `bbb-group` へ所属させ、各サービスへ許可設定をする。

まずは、各サービスに対して、acl プラグインを有効化する。

```sh
$ curl -i -X POST http://localhost:8001/services/aaa-api/plugins \
> --data "name=acl" \
> --data "config.whitelist=aaa-group"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:41:16 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 217

{
  "created_at": 1540539676065,
  "config": {
    "whitelist": [
      "acl-group"
    ],
    "hide_groups_header": false
  },
  "id": "8625c0cf-ad86-4416-b3f9-08cfb4224fe4",
  "service_id": "b371024b-cebf-4065-a1bf-58ed9aad8dce",
  "name": "acl",
  "enabled": true
}

$ curl -i -X POST http://localhost:8001/services/bbb-api/plugins \
> --data "name=acl" \
> --data "config.whitelist=bbb-group"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:41:52 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 217

{
  "created_at": 1540539712346,
  "config": {
    "whitelist": [
      "bbb-group"
    ],
    "hide_groups_header": false
  },
  "id": "d01dfc1e-5a60-4cf5-a8d5-a4d2dd974a4a",
  "service_id": "1d85e93c-8bc7-46f8-afed-a864ea8e0340",
  "name": "acl",
  "enabled": true
}
```

各 consumer にグループを設定する。

```sh
$ curl -i -X POST http://localhost:8001/consumers/aaa/acls --data "group=aaa-group"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:49:50 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 146

{
  "group": "aaa-group",
  "created_at": 1540540190325,
  "id": "32928438-8d23-4efe-b457-d735be430f7e",
  "consumer_id": "3cb5f512-ad07-4f27-9aed-4ddff00a020e"
}


$ curl -i -X POST http://localhost:8001/consumers/bbb/acls --data "group=bbb-group"
HTTP/1.1 201 Created
Date: Fri, 26 Oct 2018 07:50:09 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 146

{
  "group": "bbb-group",
  "created_at": 1540540209260,
  "id": "32db4c2b-2c2e-4f20-b869-d9e784fd8a6b",
  "consumer_id": "9f8e8674-8d0d-43c9-8c58-59331949e0bd"
}
```

Consumer `aaa` が API a にはアクセスでき、API b にはアクセスできないことを検証する。

```sh
$ curl -i -X GET http://localhost:8000/aaa --header "apikey:aaaaaaaaa"
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 10750
Connection: keep-alive
Server: openresty/1.13.6.2
Date: Fri, 26 Oct 2018 07:52:40 GMT
Etag: W/"29fe-zRGDbSTAzeA2BElashPm2g"
Vary: Accept-Encoding
Via: kong/0.14.1
X-Kong-Upstream-Status: 200
X-Kong-Upstream-Latency: 288
X-Kong-Proxy-Latency: 587
Kong-Request-ID: b85769d55d4927681238e50b07b6ba8c

<!DOCTYPE html><html><head><meta charset="utf-8"><title>Mockbin by Mashape</title><meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no"><link rel="stylesheet" type="text/css" href="https://cdnjs.cloudflare.com/ajax/libs/twitter-bootstrap/3.3.4/css/bootstrap.min.css" media="all"><link rel="stylesheet" type="text/css" href="https://fonts.googleapis.com/css?family=Open+Sans:400,600|Source+Code+Pro:200,300,400,500,600,700,900" media="all"><link rel="stylesheet" type="text/css" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.3.0/css/font-awesome.css" media="all"><link rel="stylesheet" type="text/css" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/8.4/styles/github.min.css" media="all"><link rel="stylesheet" type="text/css" href="/static/main.css" media="all"><script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/jquery/2.1.3/jquery.min.js"></script><script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/twitter-bootstrap/3.3.4/js/bootstrap.min.js"></script><script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/8.4/highlight.min.js"></script><script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/zeroclipboard/2.2.0/ZeroClipboard.min.js"></script><meta http-equiv="X-UA-Compatible" content="IE=edge"><meta name="robots" content="index,follow"><meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no"><meta itemprop="name" content="Mockbin by Mashape"><meta property="og:title" content="Mockbin by Mashape"><meta name="twitter:title" content="Mockbin by Mashape"><link rel="author" href="https://www.mashape.com/"><meta name="author" content="Mashape"><meta name="twitter:creator" content="@Mashape"><meta name="description" content="Mockbin allows you to generate custom endpoints to test, mock, and track HTTP requests &amp; responses between libraries, sockets and APIs. Made with Love by Mashape."><meta itemprop="description" content="Mockbin allows you to generate custom endpoints to test, mock, and track HTTP requests &amp; responses between libraries, sockets and APIs. Made with Love by Mashape."><meta property="og:description" content="Mockbin allows you to generate custom endpoints to test, mock, and track HTTP requests &amp; responses between libraries, sockets and APIs. Made with Love by Mashape."><meta name="twitter:description" content="Mockbin allows you to generate custom endpoints to test, mock, and track HTTP requests &amp; responses between libraries, sockets and APIs. Made with Love by Mashape."><meta itemprop="image" content="https://mockbin.org/public/share.png"><meta property="og:image" content="https://mockbin.org/public/share.png"><meta name="twitter:image:src" content="https://mockbin.org/public/share.png"><meta name="twitter:card" content="summary_large_image"><meta name="twitter:site" content="@mashape"><meta name="twitter:domain" content="mockbin.org"><link rel="canonical" href="http://mockbin.org/"><meta name="twitter:url" content="http://mockbin.org/"><meta property="og:url" content="http://mockbin.org/"><meta property="og:type" content="website"><meta property="og:site_name" content="Mockbin by Mashape"><meta property="fb:admins" content="227304446"><meta property="fb:admins" content="576641408"><link rel="shortcut icon" href="/public/favicon.ico"><link rel="icon" type="image/x-icon" href="/public/favicon.ico"><link rel="stylesheet" type="text/css" href="https://fonts.googleapis.com/css?family=Open+Sans:300italic,400italic,600italic,700italic,800italic,400,800,700,600,300|Source+Code+Pro:200,300,400,500,600,700,900" media="all"><link rel="stylesheet" type="text/css" href="/public/style.css" media="all"><meta name="google-site-verification" content="OIx3DxcNRJ_Kyd7hAtGRhZnggKpv6DRWutY7Ih9R3Ww"></head><body><header><nav class="navbar navbar-default"><div class="container"><div class="navbar-header"><button type="button" data-toggle="collapse" data-target="#navbar" class="navbar-toggle collapsed"><span class="icon-bar"></span><span class="icon-bar"></span><span class="icon-bar"></span></button><div class="navbar-brand logo"><span><a href="/"><span class="logo fa fa-terminal"></span> mockbin</a> <span class="text-muted">by <a href="https://www.mashape.com">Mashape</a></span></span></div></div><div id="navbar" class="collapse navbar-collapse"><ul class="nav navbar-nav navbar-right"><li><a href="/docs">Docs</a></li><li><a href="/bin/create">Create Bin</a></li><li><a href="https://github.com/Mashape/mockbin">Github</a></li></ul></div></div></nav></header><div class="home"><div class="showcase"><div class="container"><h1>Mockbin</h1><p class="col-lg-offset-2 col-lg-8 lead">Mockbin allows you to generate <a href="/bin/create">custom endpoints</a> to test, mock, and track HTTP requests &amp; responses between libraries, sockets and APIs.</p></div></div><div class="container"><div class="btn-toolbar"><a href="/bin/bbe7f656-12d6-4877-9fa8-5cd61f9522a9/view" class="btn btn-primary">View Sample Bin</a><a href="/bin/create" class="btn btn-success">Create Bin</a><a href="#example" class="btn btn-primary hidden-xs">Send a Request</a></div><hr><h2 class="text-center">Feature Highlights</h2><div class="row features"><div class="col-md-6"><div class="media"><div class="media-left"><img src="/public/friendly.png" class="media-object"></div><div class="media-body"><h4 class="media-heading">Mock Custom Endpoints</h4><p>Mock custom endpoints using any <a href="https://ahmadnassri.github.io/har-resources/" target="_blank">HTTP Archive (HAR)</a> response object <em>(can be used as webhooks, api mocks, or anything you want!)</em></p><p><a href="/docs">Learn More <span class="fa fa-angle-right"></span></a></p></div></div><div class="media"><div class="media-left"><img src="/public/formats.png" class="media-object"></div><div class="media-body"><h4 class="media-heading">JSON, XML, YAML, HTML</h4><p>Don't like JSON? No problem! Mockbin supports output in JSON, YAML and XML, as well as an HTML view for in-browser testing</p><p><a href="/docs#content-negotiation">Learn More <span class="fa fa-angle-right"></span></a></p></div></div><div class="media"><div class="media-left"><img src="/public/history.png" class="media-object"></div><div class="media-body"><h4 class="media-heading">Log and Inspect Calls</h4><p>Log and inspect incoming calls to your custom endpoints <em>(get detailed view to how clients are calling your api/webhook)</em></p><p><a href="/docs">Learn More <span class="fa fa-angle-right"></span></a></p></div></div></div><div class="col-md-6"><div class="media"><div class="media-left"><img src="/public/mock.png" class="media-object"></div><div class="media-body"><h4 class="media-heading">Custom HTTP Method</h4><p>No longer are you limited to <code>GET</code> &amp; <code>POST</code>, Mockbin accepts all standard Methods and allows method overriding</p><p><a href="/docs#http-methods">Learn More <span class="fa fa-angle-right"></span></a></p></div></div><div class="media"><div class="media-left"><img src="/public/inspect.png" class="media-object"></div><div class="media-body"><h4 class="media-heading">CORS Headers</h4><p>Debug your front-end JavaScript HTTP calls from any domain, Mockbin will dynamically generate Cross-Origin resource sharing headers</p><p><a href="/docs">Learn More <span class="fa fa-angle-right"></span></a></p></div></div><div class="media"><div class="media-left"><img src="/public/har.png" class="media-object"></div><div class="media-body"><h4 class="media-heading">HTTP Archive (HAR)</h4><p>Mockbin relies on the popular <a href="https://ahmadnassri.github.io/har-resources/" target="_blank">HTTP Archive (HAR)</a> format to create mock endpoints (Bins), import data and describe HTTP call logs.</p><p><a href="/docs">Learn More <span class="fa fa-angle-right"></span></a></p></div></div></div></div><hr><h2 class="text-center">Test using your preferred language:</h2><iframe id="example" src="https://api.apiembed.com/?source=http://mockbin.org/public/samples/request.json&amp;targets=all" frameborder="0" scrolling="no" marginheight="0" marginwidth="0" width="100%" height="500" seamless class="embed"></iframe></div></div><footer class="hidden-xs"><nav class="navbar navbar-default navbar-fixed-bottom"><div class="container"><div class="navbar-text"><a href="https://github.com/Mashape/mockbin" data-icon="octicon-star" data-count-href="/Mashape/mockbin/stargazers" data-count-api="/repos/Mashape/mockbin#stargazers_count" class="github-button">Star</a><span>&nbsp;</span><a href="https://github.com/Mashape/mockbin" data-icon="octicon-eye" data-count-href="/Mashape/mockbin/watchers" data-count-api="/repos/Mashape/mockbin#subscribers_count" class="github-button">Watch</a><span>&nbsp;</span><a href="https://github.com/Mashape/mockbin/issues" data-icon="octicon-issue-opened" data-count-api="/repos/Mashape/mockbin#open_issues_count" class="github-button">Issue</a></div><div class="nav navbar-right navbar-text hidden-xs"><a href="https://twitter.com/share" data-url="http://mockbin.org" data-via="Mashape" data-related="Mashape" data-hashtags="Mock, Test, Track, HTTP" data-dnt="true" class="twitter-share-button">Tweet</a><span>&nbsp;</span></div></div></nav></footer><script type="text/javascript" id="twitter-wjs" src="https://platform.twitter.com/widgets.js" async defer></script><script type="text/javascript" id="github-bjs" src="https://buttons.github.io/buttons.js" async defer></script><script type="text/javascript" src="//mashape.github.io/notification-bar/embed.js" async defer></script><script type="text/javascript">!function(){var analytics=window.analytics=window.analytics||[];if(!analytics.initialize)if(analytics.invoked)window.console&&console.error&&console.error("Segment snippet included twice.");else{analytics.invoked=!0;analytics.methods=["trackSubmit","trackClick","trackLink","trackForm","pageview","identify","group","track","ready","alias","page","once","off","on"];analytics.factory=function(t){return function(){var e=Array.prototype.slice.call(arguments);e.unshift(t);analytics.push(e);return analytics}};for(var t=0;t<analytics.methods.length;t++){var e=analytics.methods[t];analytics[e]=analytics.factory(e)}analytics.load=function(t){var e=document.createElement("script");e.type="text/javascript";e.async=!0;e.src=("https:"===document.location.protocol?"https://":"http://")+"cdn.segment.com/analytics.js/v1/"+t+"/analytics.min.js";var n=document.getElementsByTagName("script")[0];n.parentNode.insertBefore(e,n)};analytics.SNIPPET_VERSION="3.0.1";
  analytics.load('tUiM2iBCz991uF4rDF0a4WSr6NEjiVuU');
  analytics.page()
}}();</script></body></html>
```

```sh
$ curl -i -X GET http://localhost:8000/bbb --header "apikey: aaaaaaaaa"
HTTP/1.1 403 Forbidden
Date: Fri, 26 Oct 2018 07:53:19 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Server: kong/0.14.1
Content-Length: 46

{
  "message": "You cannot consume this service"
}
```

アクセス制御が効いていることを確認できた。

## 関連

- [API アグリゲーション： Kong - momota.txt](http://momota.github.io/blog/2018/10/26/kong/)
