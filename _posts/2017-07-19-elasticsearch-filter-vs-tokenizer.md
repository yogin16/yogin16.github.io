---
layout:     post
title:      "Elasticsearch: Filter vs Tokenizer"
date:       2017-07-19 01:28:58 +0530
comments:   true
---
I recently learned difference between `mapping` and `setting` in Elasticsearch. Which I wish I should have known earlier. Along the way I understood the need for `filter` and difference between `filter` and `tokenizer` in setting.

Most of the time it is very important to get the mapping and setting of an index right before we are configuring or even start using ES. Which analyzers to use, which fields to index, how many shards; such questions have to be thought upon before we hit production with any ES index. An error here would cost more in term of re-indexing the data, or in-efficient use of memory or other resources.
Elasticsearch has fairly detailed [documentation](https://www.elastic.co/guide/en/elasticsearch/guide/current/_index_settings.html) on everything and that explained that the index setting defines the configuration of the `index` and customises analyzers. And mapping is for the `type` and defines the schema and which of the analyzers to be used on a property.

## analysis
> Analysis is the process of converting text, like the body of any email, into tokens or terms which are added to the inverted index for searching.

In index setting when we define [analysis](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) , we have to define analyzers (which can be standard or custom) and tokenizers (which can be standard or custom). This will define how our indexing would behave. An analyzer will have `type`, must have (only one) `tokenizer` & can have additionally (one or many in order) `filter`, `char_filter`.

## filter vs tokenizer

`filters` would apply **after** `tokenizer` on tokens. Classic example for the use case would be `lowecase` filter or `stop` filter to remove the terms which are in stop words, like `the`.
This all sounds ok and basic, but can lead to mistakes in configuration. For example, taking, `NGram` tokenizer and filter example as explained in [this book](http://exploringelasticsearch.com/searching_usernames_and_tokenish_text.html) to support "search as you type" feature.
Let say in our analyzer we want to use `Ngrams` for supporting search even when user enters incomplete word.

#### create test index1
```json
curl -XPUT 'localhost:9200/test_ngram_1?pretty' -H 'Content-Type: application/json' -d '{
    "settings": {
        "index": {
            "analysis": {
                "analyzer": {
                    "customNgram": {
                        "type": "custom",
                        "filter": "lowercase",
                        "tokenizer": "customNgram"
                    }
                },
                "tokenizer": {
                    "customNgram": {
                        "min_gram": "3",
                        "type": "edgeNGram",
                        "max_gram": "18",
                        "side": "front"
                    }
                }
            }
        }
    }
}
```

This reads as - it would tokenize with `edgeNGram` and lowecase them.

This analyzer works well when you have search on something like `username` - where it is single token. But fails when you have to search on something like a message test, which is more like a sentence than a word.

#### analyze test index1
```json
curl -XPOST localhost:9200/test_ngram_1/_analyze -d '{
    "analyzer": "spr_ngram",
    "text": "Quick Fox"
}'
```

Result:

```javascript
{
    "tokens": [{
        "token": "qui",
        "start_offset": 0,
        "end_offset": 3,
        "type": "word",
        "position": 0
    }, {
        "token": "quic",
        "start_offset": 0,
        "end_offset": 4,
        "type": "word",
        "position": 1
    }, {
        "token": "quick",
        "start_offset": 0,
        "end_offset": 5,
        "type": "word",
        "position": 2
    }, {
        "token": "quick ",
        "start_offset": 0,
        "end_offset": 6,
        "type": "word",
        "position": 3
    }, {
        "token": "quick f",
        "start_offset": 0,
        "end_offset": 7,
        "type": "word",
        "position": 4
    }, {
        "token": "quick fo",
        "start_offset": 0,
        "end_offset": 8,
        "type": "word",
        "position": 5
    }, {
        "token": "quick fox",
        "start_offset": 0,
        "end_offset": 9,
        "type": "word",
        "position": 6
    }]
}
```

Notice: whenever someone search with `fox` this wouldn't qualify, as the tokens are just one sentence. We have never used "whitespace" tokenizer in analysis.

Lets fix that from what we know of using `filters`.

#### create test index2:
```json
curl -XPUT 'localhost:9200/test_ngram_2?pretty' -H 'Content-Type: application/json' -d'{
    "settings": {
        "index": {
            "analysis": {
                "analyzer": {
                    "customNgram": {
                        "type": "custom",
                        "tokenizer": "whitespace",
                        "filter": ["lowercase", "customNgram"]
                    }
                },
                "filter": {
                    "customNgram": {
                        "type": "edgeNgram",
                        "min_gram": "3",
                        "max_gram": "18",
                        "side": "front"
                    }
                }
            }
        }
    }
}'
```

Now, our tokenizer is not ngram. our tokenizer is `whitespace`. The `ngram` is part of filter. which would be applied after tokenizer, and on tokens.

#### analyze test index2

```json
curl -XPOST localhost:9200/test_ngram_2/_analyze -d '{
    "analyzer": "spr_ngram",
    "text": "Quick Fox"
}'
```

Result:
```javascript
{
    "tokens": [{
        "token": "qui",
        "start_offset": 0,
        "end_offset": 5,
        "type": "word",
        "position": 0
    }, {
        "token": "quic",
        "start_offset": 0,
        "end_offset": 5,
        "type": "word",
        "position": 0
    }, {
        "token": "quick",
        "start_offset": 0,
        "end_offset": 5,
        "type": "word",
        "position": 0
    }, {
        "token": "fox",
        "start_offset": 6,
        "end_offset": 9,
        "type": "word",
        "position": 1
    }]
}
```

This now qualifies when searching `fox`.

## References
1. [http://exploringelasticsearch.com/searching_usernames_and_tokenish_text.html](http://exploringelasticsearch.com/searching_usernames_and_tokenish_text.html)
1. [https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer-anatomy.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer-anatomy.html)
1. [https://www.elastic.co/guide/en/elasticsearch/reference/5.4/analysis-edgengram-tokenfilter.html#analysis-edgengram-tokenfilter](https://www.elastic.co/guide/en/elasticsearch/reference/5.4/analysis-edgengram-tokenfilter.html#analysis-edgengram-tokenfilter)
1. [https://www.elastic.co/blog/found-text-analysis-part-1](https://www.elastic.co/blog/found-text-analysis-part-1)
