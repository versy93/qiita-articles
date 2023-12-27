---
title: Elasticsearchでインデックス時、検索時にkuromojiの形態素解析を適用する
tags:
  - Elasticsearch
  - kuromoji
private: false
updated_at: '2021-12-13T16:06:50+09:00'
id: f5b8b5edf0dfc8db8280
organization_url_name: null
slide: false
ignorePublish: false
---

Elasticsearchサーバのドメインとポートは自身の環境に合わせて変更する。




# インデックスを閉じる

elasticsearchのインデックスには「閉じる」という概念がある。

analyzerの定義変更の際には「閉じて開く」必要がある



## 実行

```
POST http://elasticsearch:9210/my_index/_close
```



## 結果

```
{
    "acknowledged": true,
    "shards_acknowledged": true,
    "indices": {
        "shop": {
            "closed": true
        }
    }
}
```





# analyzerを変更



## 実行



`default` インデックス登録時と検索時両方で使用されるよう設定

`default_index` インデックス登録時のみ使用されるよう設定

`default_search` 検索時のみ使用されるよう設定



```
PUT http://elasticsearch:9210/my_index/_settings
{
    "analysis": {
        "analyzer": {
            "default": {
                "tokenizer": "kuromoji_tokenizer"
            }
        }
    }
}
```



## 結果

```
{
    "acknowledged": true
}
```





# インデックスを開ける

説明は省略



## 実行

```
POST http://elasticsearch:9210/my_index/_open
```



## 結果

```
{
    "acknowledged": true,
    "shards_acknowledged": true
}
```





# 確認



## 実行

```
GET ttp://0.0.0.0:9210/my_index/_settings
```



## 結果

```
{
    "my_index": {
        "settings": {
            "index": {
                "number_of_shards": "1",
                "provided_name": "my_index",
                "creation_date": "0000000000000",
                "analysis": {
                    "analyzer": {
                        "my_analyzer": {
                            "type": "custom",
                            "tokenizer": "kuromoji_tokenizer"
                        }
                    }
                },
                "number_of_replicas": "1",
                "uuid": "xxxxwixxxxu6rxxxB6Lxxx",
                "version": {
                    "created": "7080099"
                }
            }
        }
    }
}
```



以上でkuromojinの形態素解析がインデックス時と検索時に適用される。
