I recently started working for Dgraph Labs. I was the first employee to join the India office,
and now, we have grown to a team of 5 engineers. And, [we are hiring](https://dgraph.io/careers).
If you are looking for challenging distributed systems work, or excited about GraphQL,
feel free to reach out to us.

One of the first issue that I solved after joining Dgraph, is an issue with Dgraph's datetime
datatype. In this article, I talk about how I discovered the issue and then, resolved it.

### Bug Discovery
While working on [Flock](https://github.com/dgraph-io/flock), I observed that timestamp
comparisons were not working correctly in Dgraph when the data or query had timezone information
in it. To show the issue, let's first quickly setup Dgraph using following commands -
```
docker run --rm -it -p 5080:5080 -p 6080:6080 -p 8080:8080 -p 9080:9080 -p 8000:8000 --name dgraph dgraph/dgraph:v1.0.14 dgraph zero
docker exec -it dgraph dgraph alpha --lru_mb 2048 --zero localhost:5080
docker exec -it dgraph dgraph-ratel
```

We will use ratel to showcase the issue. To open ratel, you could go the URL http://localhost:8000
in your browser. Now, let's setup the schema -
```
created_at : datetime @index(hour) .
```

Now, perform following mutations to add data to Dgraph -
```
{
  set {
    _:user1 <created_at> "2019-03-28T14:41:57-06:00" .
    _:user1 <author_name> "Rahul" .
    _:user2 <created_at> "2019-03-28T18:40:57+01:00" .
    _:user2 <author_name> "Ashish" .
  }
}
```

Now, run following query -
```
{
  tweets(func: gt(created_at, "2019-03-28T15:00:00+00:00")) {
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
      "parsing_ns": 10222,
      "processing_ns": 11203368,
      "encoding_ns": 681271
    },
    "txn": {
      "start_ts": 10
    }
  },
  "data": {
    "tweets": [
      {
        "created_at": "2019-03-28T18:40:57+01:00",
        "uid": "0x2"
      }
    ]
  }
}
```

The first user has `created_at` timestamp `2019-03-28T14:41:57-06:00` which is equal to
`2019-03-28T20:41:57+00:00` and bigger than the timestamp in the query `2019-03-28T15:00:00+00:00`
but still doesn't show up in the response of the query. Before I explain how we fixed this
issue, let me first explain how indexes work in Dgraph.


### Datetime Indexes in Dgraph
Dgraph has indexes available for various types including datetime type. All the indexes
are stored in [badger](https://github.com/dgraph-io/badger), an embedded key value store
written purely in Go. Dgraph generates one or more tokens for each input value and stores
a reference to the data (value in badger) while keys (key in badger) being the tokens generated.
All such tokens generated from all values are stored in sorted manner so that they can be
searched through quickly while executing a query.

For a Datetime index, Dgraph allows hour, day, month and year index. The granularity of
datetime index should be carefully chosen in order to reduce linear scan after index lookup.
For example, if we create a day index, and there is a lot of data corresponding to a each
hour, every time we do comparisons, we will have to linearly look up the data corresponding
to a particular day to find the timestamp that we are looking for. On the other hand, if we
have little data for each hour, the index size will be unnecessarily large, leading to
fragmentation of the data in badger, resulting in large scan time (for greater or less than
queries, for example).

### Time Package
Now, in order to create the datetime index, we use the `Hour()`, `Day()`, `Month()` and
`Year()` functions of the `time` package to generate tokens
(https://github.com/dgraph-io/dgraph/blob/master/tok/tok.go). All these functions internally
call the `abs()` function in order to compute the final results.
```go
// abs returns the time t as an absolute time, adjusted by the zone offset.
// It is called when computing a presentation property like Month or Hour.
func (t Time) abs() uint64 {
```

The `abs()` function returns number of seconds since the zero time accounting for the
timezone information.
```go
if l != &utcLoc {
  if l.cacheZone != nil && l.cacheStart <= sec && sec < l.cacheEnd {
    sec += int64(l.cacheZone.offset)
  } else {
    _, offset, _, _ := l.lookup(sec)
    sec += int64(offset)
  }
}
```

Hence, for timestamp `2019-03-28T14:41:57+06:00`, the `Hour()` function will return the
value `14` and for timestamp `2019-03-28T14:41:57+00:00`, the `Hour()` function will still
return the same value `14`. Due to this behavior, the index is not correctly built.

### Root Cause
Now, in our example above, the data has two timestamps 1)`2019-03-28T14:41:57-06:00` and
2) `2019-03-28T18:40:57+01:00` and we have created `hour` index on the `created_at`
predicate. The `hour` index will **incorrectly** store roughly following information -
```
14 -> 0x01
18 -> 0x02
```

The query is trying to find out all users having `created_at` timestamp bigger than
`2019-03-28T15:00:00+00:00`. While processing the query, Dgraph will compute the same
`hour` tokenizer for this timestamp to be `15`. Then, it will scan through all the
users in the index having the token value bigger than `15` and only return the user
with uid `0x02`.

### Fixing Bugs
Once the issue was clear, the fix was pretty straight forward. Instead of building tokens
directly on the given timestamp, we first covert the timestamp into `UTC()` timezone and
then compute various tokens. This normalizes all timestamps into single timezone and the
`Hour()`, `Day()`, `Month()` and `Year()` information can be correctly compared. The PR
for this fix is available [here](https://github.com/dgraph-io/dgraph/pull/3251).

Once this change is included, the index will look like this instead -
```
17 -> 0x02
20 -> 0x01
```

The timestamp in the query will generate token value as `15` and while scanning through
the index, we will see both the users. The correct response will, therefore, look like -
```
{
  "data": {
    "tweets": [
      {
        "created_at": "2019-03-28T14:41:57-06:00",
        "uid": "0x1"
      },
      {
        "created_at": "2019-03-28T18:40:57+01:00",
        "uid": "0x2"
      }
    ]
  },
  "extensions": {
    "server_latency": {
      "parsing_ns": 16424,
      "processing_ns": 12946317,
      "encoding_ns": 784262
    },
    "txn": {
      "start_ts": 13
    }
  }
}
```

You can see that both the users are included in this response.
