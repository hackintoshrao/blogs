I recently started working for Dgraph Labs. I was the first employee to join the India office,
and now, we have grown to a team of 5 engineers. And, [we are hiring](https://dgraph.io/careers).
If you are looking for challenging distributed systems work, or excited about GraphQL,
feel free to reach out to us.

One of the first issue that I solved after joining Dgraph is an issue with datetime datatype.
In this article, I talk about how I discovered the issue and then, resolved it.

### Bug Discovery
While working on [Flock](https://github.com/dgraph-io/flock), I observed that timestamp
comparisions were not working correctly in Dgraph when the data or query had timezone
information in it. Consider the following example query.

Let's quickly setup Dgraph using following commands -
```
docker run --rm -it -p 5080:5080 -p 6080:6080 -p 8080:8080 -p 9080:9080 -p 8000:8000 --name dgraph dgraph/dgraph:v1.0.14 dgraph zero
docker exec -it dgraph dgraph alpha --lru_mb 2048 --zero localhost:5080
docker exec -it dgraph dgraph-ratel
```

We will use ratel to showcase the issue. First, let's setup the schema -
```
created_at : datetime @index(hour) .
```

Now, perform following mutations to add data to Dgraph -
```
{
  set {
    _:tweet1 <created_at> "2019-03-28T14:41:57+06:00" .
    _:tweet1 <author_name> "Rahul" .
    _:tweet2 <created_at> "2019-03-28T14:40:57" .
    _:tweet2 <author_name> "Ashish" .
  }
}
```

Now, run following query -
```
{
  tweets(func: gt(created_at, "2019-03-28T14:30:57")) {
    created_at
    uid
  }
}
```

This is the response we get -
```
{
  "extensions": {
    "server_latency": {
      "parsing_ns": 14202,
      "processing_ns": 11075948,
      "encoding_ns": 703296
    },
    "txn": {
      "start_ts": 321
    }
  },
  "data": {
    "tweets": [
      {
        "uid": "0x2",
        "author_name": "Ashish",
        "created_at": "2019-03-28T14:41:57Z"
      }
    ]
  }
}
```

The first user has `created_at` timestamp `2019-03-28T14:41:57+06:00` which is equal to
`2019-03-28T08:41:57+00:00` and bigger than the timestamp in the query `2019-03-28T14:30:57`
but still doesn't show up in the resonse of the query. Before I explain how we fixed this
issue, let me first explain how indexes work in Dgraph.


### Datetime Indexes in Dgraph
Dgraph has indexes available for various types including datettime type. All the indexes
are stored in badger, an embedded key value store written purely in Go. Dgraph generates
one or more tokens for each input value and stores a reference to the data (value in badger)
while keys (key in badger) being the tokens generated. All such tokens generated from all
values are stored in sorted manner so that they can be searched through quickly while
executing a query.

For a Datetime index, Dgraph allows hour, day, month or year index. The granularity of
datetime index should be carefully chosen in order to reduce linear scan after index lookup.
For example, if we create a day index, and there is a lot of data corresponding to a each
hour, everytime we do comparision, we will have to linearly look up the data corresponding
to a particular which could have been reduced by using hour granularity of the index.

### Time Package
Now, in order to create the index, we use the `Hour()`, `Day()`, `Month()` and `Year()`
functions of the `time` package.

### Fixing Bugs
