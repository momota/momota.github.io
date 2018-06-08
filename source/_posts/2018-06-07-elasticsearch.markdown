---
layout: post
title: "docker で Elasticsearch"
date: 2018-06-07 13:56
comments: true
categories: elasticsearch docker
---
Docker for Windows で Windows10 上に Elasticsearch を動かしたときのメモ。

- Docker: 18.03.1-ce-win65 (17513)
- Elasticsearch: 6.2.4


<!-- more -->


日本語を扱いたいのでプラグイン `kuromoji` をインストールする Dockerfile を用意する。

Dockerfile の内容は以下。

```
FROM docker.elastic.co/elasticsearch/elasticsearch:6.2.4

# Plugin x-pack already exists in this image
# RUN elasticsearch-plugin install --batch x-pack
RUN elasticsearch-plugin install analysis-kuromoji
```

この Dockerfile に基づいてイメージをビルドする。

```
> docker build -t es-kuromoji:1.0 ./
> docker images
REPOSITORY                                               TAG                 IMAGE ID            CREATED              SIZE
es-kuromoji                                              1.0                 dxxxxxxxxxxx        About a minute ago   519MB
...
```

[Install Elasticsearch with Docker | Elasticsearch Reference ［6.2］ | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run) に書いてあるとおりに起動する。

```
> docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" es-kuromoji:1.0
```

起動を確認する。

```
> docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                                            NAMES
xxxxxxxxxxxx        es-kuromoji:1.0     "/usr/local/bin/dock…"  About a minute ago   Up Ab out a minute  0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp   sharp_hypatia
```


Elasticsearch の正常稼働を REST APIから確認する。 Ubuntu (WSL) から `curl` を使う。

`health` が `green` なので問題ない。


```sh
# In my environment, need `noproxy` option: curl http://localhost:9200/_cat/indices?v --noproxy localhost
$ curl http://localhost:9200/_cat/indices?v
health status index                       uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .monitoring-es-6-2018.06.07 xxxxxxxxxxxxxxxxxxxxxx   1   0       3094            2      1.1mb          1.1mb
```

Windows側で確認する場合は、`curl` の代わりに PowerShell `Invoke-WebRequest` コマンドで確認できる。

```sh
# In windows, we can confirm Elasticsearch is running by using powershell
PS C:\> Invoke-WebRequest -Uri http://localhost:9200/_cat/indices?v

StatusCode        : 200
StatusDescription : OK
Content           : health status index                       uuid                   pri rep docs.count docs.deleted store.size pri.store.size
                    green  open   .monitoring-es-6-2018.06.07 xxxxxxxxxxxxxxxxxxxxxx   1   0     ...
RawContent        : HTTP/1.1 200 OK
                    Content-Length: 246
                    Content-Type: text/plain; charset=UTF-8

                    health status index                       uuid                   pri rep docs.count docs.deleted store.size pri.store.s...
Forms             : {}
Headers           : {[Content-Length, 246], [Content-Type, text/plain; charset=UTF-8]}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 246
```


index を作ってみる。
index は RDB のテーブルやビューにあたる。

PUT メソッドを使って `customer` という index を作る。
`pretty` は JSON を pretty-print (pretty-print) してくれる。 `jq` みたいに整形してくれる。

```sh
$ curl -X PUT http://localhost:9200/customer?pretty
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "customer"
}
```

index が増えていることが確認できる。

```sh
$ curl http://localhost:9200/_cat/indices?v
health status index                       uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .monitoring-es-6-2018.06.07 xxxxxxxxxxxxxxxxxxxxxx   1   0       4775           44      1.9mb          1.9mb
yellow open   customer                    xxxxxxxxxxxxxxxxxxxxxx   5   1          0            0      1.1kb          1.1kb
```

document を追加する。
RDB のレコードみたいなもの。

```sh
$ curl -H "Content-Type: application/json" -X PUT http://localhost:9200/customer/_doc/1?pretty -d '
> {
>   "name": "John Doe"
> }
> '
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```

PowerShell バージョンは以下。

```sh
$person = @{
  "name": "John Doe"
}
$json = $person | ConvertTo-Json
$response = Invoke-RestMethod 'http://localhost:9200/customer/_doc/1' -Method Put -Body $json -ContentType 'application/json'
```

document の確認。

```sh
$ curl -sS -w '\n' http://localhost:9200/customer/_doc/1?pretty
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "name" : "John Doe"
  }
}
```
