---
layout: post
title:  "Conditional Aggregation with Couchbase Server Views"
date:   2015-06-23 17:57:36
categories: views
---

Recently I became interested in the possibility of extracting regular time-based reports across a dataset.

The purpose of the reports is to give a snapshot of outcome of a series of  ‘events’ that were each tagged with a timestamp and a number of fields which could satisfy a variety of conditions.

As an example of the sort of data, consider the following document:

```javascript
var json = { 
"player_id": "simon",
"time": "2015-06-01T14:15:57.455Z",
"club": "long",
"shot_type": "fade",
"outcome": "success"
}
```

`club` could be either long or short.
`shot_type` could be fade, draw.
`outcome` could be success or failure according to some pre-defined criteria.

When designing views it is always best to start by considering what question you are asking and in what form you wish the answer to be presented. The sort of question that we want to answer is:

'**_For each player, between 14:00 to 15:00, how many shots were made with each type of club, how many shots were of each type, how many shots resulted in each outcome?_**'

How shall we achieve this?

1. If we want to eventually group by player, we are going to need to emit the player_id as part of our key emitted by our map function.

2. We want to group by time ranges so we will also need to make use of the `dateToArray()` function that is available in the Couchbase map-reduce framework in order to allow grouping across time ranges with variable precision at query time. 
In order to perform the grouping at query time, we will need the array created by this function to be part of the document key emitted by our map function.

3. Finally, as soon as we see that we want some sort of aggregation to be the value connected with each emitted key, we should associate this with a requirement for a **reduce** function.

Plenty of documentation exists on [docs.couchbase.com] which explains how to use the built in `_sum`, `_count` and `_stats` reduce functions that ship with Couchbase Server.

Since we have a variety of conditions which we wish to evaluate and aggregate, we cannot use a built in reduce, as they are designed for emit statements that emit single values.

We can get some inspiration on how to achieve our goal by taking a look at the source javascript for the built in `_sum` function:

```javascript
var _sum = function(key, values, rereduce) {
  var sum = 0;
  for(i=0; i < values.length; i++) {
    sum = sum + values[i];
  }
  return(sum);
}
```

This demonstrates to us that the input to the reduce function at each step will be the key and an array of values, which are the documents emitted by the map function (or emitted by a previous reduce step, since we might be re-reducing).

There is nothing therefore to stop us adapting this sum function to expect an input in the form of a JSON document, and accessing its fields in order to perform our summation.

How then, will we create this JSON? The answer is to do some conditional evaluation in our map function based on our query requirements and to finally emit a JSON document which indicates whether the criteria we are interested in was met by the various fields in that document.

Here’s what the map function looks like in our case:

```javascript
var map = function (doc, meta) {
  var dateArray = dateToArray(doc.time);
  var group = [doc.player_id].concat(time);
  
  emit(group, {
    "total": 1,
    "short": doc.club == "short" ? 1 : 0,
    "long": doc.club == "long" ? 1 : 0,
    "fade": doc.shot_type == "fade" ? 1 : 0,
    "draw": doc.shot_type == "draw" ? 1 : 0,
    "success": doc.outcome == "success" ? 1 : 0,
    "failure": doc.outcome == "failure" ? 1 : 0
  });
}
```
So: we build a JSON object which contains the fields that we are interested in summing over and for each item to be aggregated we assign a Boolean value to it which indicates whether that condition was true or false.

The other important thing to note here is how we build our key. We are emitting a key which is an array. The `player_id` comes first, followed by the date information, in increasing order of granularity. This will let us use the `group_level` parameter in our query to decide on the player and period we want to query. 

Returning to our reduce function, we need to adapt the built in sum function to work with our JSON objects.

This is achieved by summing over each field, and it looks like this:

```javascript
var reduce = function(key, values, rereduce) {
  var agg = {
    "total": 0,
    "short": 0,
    "long": 0,
    "fade": 0,
    "draw": 0,
    "success": 0,
    "failure": 0
  };
  
  for(var i = 0, len = values.length; i < len; i++) {
    var val = values[i];
    
    for (var prop in val) {
      agg[prop] += val[prop]
    }
  }
  
  return agg;
}
```

Now we can query via REST or SDK and use the `group_level` parameter to choose the period over which to get our aggregated results!

[docs.couchbase.com]:      http://docs.couchbase.com

