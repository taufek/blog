---
layout: post
title:  "ElasticSearch Geo Shape"
date:   2020-06-17 00:58:00 +0800
categories: ruby
hero_src: mobile_app.jpg
---

In previous [post]({% post_url 2020-06-16-elasticsearch-with-ruby %}), we looked
at quick start guide on how to use ElasticSearch with Ruby.

In this post, we will look at how to search based on geo location. Lets say
we have food delivery app and we would like to find out the delivery surcharge
amount for certain area.

## Create Index with Geo Shape Mapping

```ruby

client = Elasticsearch::Client.new

mapping = {
  surcharge: {
    properties: {
      delivery_area: {
        type: 'geo_shape',
      }
    }
  }
}

client.create(index: :surcharges, type: :surcharge, body: {})

client.indices.put_mapping(index: :surcharges, type: :surcharge, body: mapping)

client.indices.get(index: :surcharges)

# Output
{
  "surcharges"=> {
    "aliases"=>{},
    "mappings"=>{
      "surcharge"=>{
        "properties"=>{
          "delivery_area"=>{"type"=>"geo_shape"}
        }
      }
    },
    "settings"=> {...}
  }
}
```

Above code, will create an index named `surcharges` and a document named
`surcharge`. It has property of `delivery_area` with `geo_shape` data type.

## Index Surcharge Document with Polygon Coordinates

Below shows polygon area that I've created with Google Maps.
![klcc_polygon](/images/klcc_polygon.png)

Below is the surcharge document which saying within that polygon area, the
surcharge amount is $10.

```ruby
surcharge = {
  id: 1,
  delivery_area: {
    type: 'polygon',
    coordinates: [[
      # longitude, latitude
      [101.710997, 3.157035],
      [101.710997, 3.152954],
      [101.717316, 3.152954],
      [101.717316, 3.157035],
      [101.710997, 3.157035],
    ]]
  },
  amount: 10.0
}

client.index(
  id: surcharge[:id],
  index: :surcharges,
  type: :surcharge,
  body: surcharge
)

# Output
{
  "_index"=>"surcharges",
  "_type"=>"surcharge",
  "_id"=>"1",
  "_version"=>1,
  "result"=>"created",
  "_shards"=>{"total"=>2, "successful"=>1, "failed"=>0},
  "_seq_no"=>1,
  "_primary_term"=>1
}
```

## Search with Geo Point

Once I have my surcharge data indexed, I can start querying based on a
coordinate. If my drop off point in somewhere inside the polygon, I'm expecting
the query will return the surcharge document.
![inside_polygon](/images/inside_polygon.png)

```ruby
query = {
  query: {
    bool: {
      filter: {
        geo_shape: {
          'delivery_area': {
            shape: {
              type: :point,
                coordinates: [
                  101.714997, #longitude
                  3.155647 #latitude
                ]
            }
          }
        }
      }
    }
  }
}


client.search(index: 'surcharges', body: query)

# Output
{
  "took"=>9,
  "timed_out"=>false,
  "_shards"=>{
    "total"=>5,
    "successful"=>5,
    "skipped"=>0,
    "failed"=>0
  },
 "hits"=>{
   "total"=>1,
   "max_score"=>1.0,
   "hits"=>[
     {
       "_index"=>"surcharges",
       "_type"=>"surcharge",
       "_id"=>"1",
       "_score"=>1.0,
       "_source"=>{
          "id"=>1,
          "delivery_area"=>{
            "type"=>"polygon",
             "coordinates"=>[[[101.710997, 3.157035], [101.710997, 3.152954], [101.717316, 3.152954], [101.717316, 3.157035], [101.710997, 3.157035]]]
          },
         "amount"=>10.0
        }
      }
    ]
  }
}
```

Then if my drop off point in somewhere outside the polygon, I'm expecting the query
will not return any surcharge document.
![outside_polygon](/images/outside_polygon.png)

```ruby
query = {
  query: {
    bool: {
      filter: {
        geo_shape: {
          'delivery_area': {
            shape: {
              type: :point,
              coordinates: [
                101.7077343, #longitude
                3.157032 #latitude
              ]
            }
          }
        }
      }
    }
  }
}


client.search(index: 'surcharges', body: query)

# Output
{
  "took"=>0,
  "timed_out"=>false,
  "_shards"=>{
    "total"=>5,
    "successful"=>5,
    "skipped"=>0,
    "failed"=>0
  },
  "hits"=>{
    "total"=>0,
    "max_score"=>nil,
    "hits"=>[]
  }
}
```

## Conclusions

Geo shape is useful when you have data associated with coordinates of land
area. Once you indexed those as a polygon in ES index, you could use Geo Shape
query to fetch documents that intersect with a particular geo point. There are
many other useful features too such as you could set a polygon with a hole in
the middle or you could search based on radius instead of point. For more info
you can head to ES official documents ([geo-shape-datatype] and [geo-shape-query]).

[geo-shape-datatype]: https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-shape.html
[geo-shape-query]: https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-shape-query.html
