---
layout: post
title:  "Learn ElasticSearch with Ruby"
date:   2020-06-16 00:00:00 +0800
categories: ruby
hero_src: elasticsearch.png
---

ElasticSearch (ES) is a popular search and analytics engine. I've been using it
in most of my Ruby on Rails projects. Usually in a project we will started of
by building an ActiveRecord model and establish the ES index mapping before we
could start running the search.

In this post, I'll start with basic and minimal code as possible to get us
started with ES.

## Pre-requisite

Install [elasticsearch] gem.

[elasticsearch]: https://github.com/elastic/elasticsearch-ruby

## Initialize ES Client

```ruby
client = Elasticsearch::Client.new
```

We'll be using above `client` instance throughout our example.

## Create ES Index Mapping

```ruby
mapping = {
  doc: {
    properties: {
      name: {
        type: :text
      },
      birth_date: {
        type: :date
      }
    }
  }
}

client.create(index: :foo, type: :doc, body: {})

client.indices.put_mapping(index: :surcharges, type: :doc, body: mapping)
```

This creates new index named `foo` with following properties:
1. `name` field as `text` data type.
2. `birth_date` field as `date` data type.

## Index Document(s)

```ruby
adam = {
  name: 'Adam Hakeem',
  birth_date: '2006-12-08'
}

alif = {
  name: 'Alif Hussain',
  birth_date: '2009-04-28'
}

ammar = {
  name: 'Ammar Hamzah',
  birth_date: '2016-07-03'
}

client.index(id: 1, index: :foo, type: :doc, body: adam)
client.index(id: 2, index: :foo, type: :doc, body: alif)
client.index(id: 3, index: :foo, type: :doc, body: ammar)
```

## Search away

Now you could start searching your data in ES index.

Below search by name with partial keyword `am*`.

```ruby
body = {
  query: {
    query_string: {
      query: 'am*'
    }
  }
}

client.search(index: :foo, body: body)

#Output:
{
  "took"=>13,
  "timed_out"=>false,
  "_shards"=>{"total"=>5, "successful"=>5, "skipped"=>0, "failed"=>0},
  "hits"=> {
    "total"=>1,
    "max_score"=>0.2876821,
    "hits"=>[
      {"_index"=>"foo", "_type"=>"doc", "_id"=>"3", "_score"=>0.2876821, "_source"=>{"name"=>"Ammar Hamzah", "birth_date"=>"2016-07-03"}}
    ]
  }
}
```

Below search by date range.

```ruby
body = {
  query: {
    range: {
      birth_date: {
        gte: '2009-01-01'
      }
    }
  }
}

client.search(index: :foo, body: body)

#Output:
{
  "took"=>2,
  "timed_out"=>false,
  "_shards"=>{"total"=>5, "successful"=>5, "skipped"=>0, "failed"=>0},
  "hits"=> {
    "total"=>2,
    "max_score"=>1.0,
    "hits"=> [
      {"_index"=>"foo", "_type"=>"doc", "_id"=>"2", "_score"=>1.0, "_source"=>{"name"=>"Alif Hussain", "birth_date"=>"2009-04-28"}},
      {"_index"=>"foo", "_type"=>"doc", "_id"=>"3", "_score"=>1.0, "_source"=>{"name"=>"Ammar Hamzah", "birth_date"=>"2016-07-03"}}
    ]
  }
}
```

## Cleanup After Yourself

If you are done playing around with your index, you might want to remove your
dummy data. You can run below to delete our particular dummy index.

```ruby
client.indices.delete(index: :foo)
```

## Conclusions

Hopefully this will give you a good start on how to learn ElasticSearch with
Ruby.
