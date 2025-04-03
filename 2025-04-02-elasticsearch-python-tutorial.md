# elasticsearch python tutorial

**Date:** 2025-04-02 23:24:54

[Pythonで作るはじめてのElasticsearchアプリケーション: Pythonで作る検索アプリケーション入門](https://www.amazon.co.jp/Python%E3%81%A7%E4%BD%9C%E3%82%8B%E3%81%AF%E3%81%98%E3%82%81%E3%81%A6%E3%81%AEElasticsearch%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3-%E6%9C%AC%E7%94%B0%E5%B4%87%E6%99%BA-ebook/dp/B082ZTXBNZ)を読んだのでその感想ややったことを書く。

## 1章
Elasticsearchは、Elastic社が提供するオープンソースの全文検索サービス
全文検索とは、コンピュータにおいて、複数の文書（ファイル）から特定の文字列を検索すること。（出典:https://ja.wikipedia.org/wiki/%E5%85%A8%E6%96%87%E6%A4%9C%E7%B4%A2）

Pythonで作成した検索アプリケーションを通して学ぶ。

Elasticsearchは高速で、かつNodeと呼ばれるサーバを増やすことでデータ量や書き込み速度を分散させることができ、スケールさせることもできる。

Elasticsearch,Kibana,Beats,Logstashの4つのコアソリューションをElastic Stackとよぶ。

### ElasticSearchのインストール
本ではインストールしていたが、ここではDockerを使ってみる。
https://hub.docker.com/_/Elasticsearch

docker-hubからpullしてくる。
```
docker pull elasticsearch
```

最新のelasticsearchをpull使用すると存在しないとエラーになった。
```
Error response from daemon: manifest for elasticsearch:latest not found: manifest unknown: manifest unknown
```

tagから最新のバージョンを調べて取得してきた。（8.17.4が最新だった）
```
docker pull elasticsearch:8.17.4
```


