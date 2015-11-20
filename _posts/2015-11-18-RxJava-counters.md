---
layout: post
title:  "Correctly Reading Couchbase Atomic Counters in RXJava"
date: 2015-11-18 00:10
categories: [couchbase, rxjava]
---

A short post to help others avoid some frustration in using a typical counter pattern when implementing code in RxJava.

Often in Couchbase one uses an atomic counter to keep track of the number of documents of a certain type.

It's easy enough to create or increment these counters:

```java

bucket.async().counter(docType + "::Count", 1, 1) 

```
When you come to read the counter though, if you try and do a standard:

```java

bucket.async().get(docType + "::Count"); 

```

The best that the SDK can know (given that we have different types of JsonDocument classes) is that the return type will be of Observable\<Document\>.

If we KNOW that we are getting a counter we should use the different method signature that allows us to specify the return class. For a counter, this is:

```java

bucket.async()
	.get(docType + "::docCount", JsonLongDocument.class)

```

We can see here that the class of the returned document type is specified. This lets us then treat the counter object correctly and extract its value for further processing.
