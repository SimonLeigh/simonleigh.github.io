---
layout: post
title:  "Modelling NOAA ISD Data in Couchbase - Part 2"
date: 2015-11-11 00:20
categories: [document design, noaa, spatial]
---

It's been some time since my last blog post so I thought that I would add a quick entry to describe the data model for the Spatial demo that I often do at meet-ups and other events.

Let's consider the NOAA ISH/ISD dataset. It includes samples of weather (temp/pressure/wind speed/etc.) for the entire planet in at least hourly intervals: 

> The Integrated Surface Database (ISD) consists of global hourly and synoptic observations compiled from numerous sources into a single common ASCII format and common data model. ISD was developed as a joint activity within Asheville's Federal Climate Complex. NCEI, with U.S. Air Force and Navy partners, began the effort in 1998 with the assistance of external funding from several sources. ISD integrates data from over 100 original data sources, including numerous data formats that were key-entered from paper forms during the 1950s–1970s time frame.

![ISD Info](https://www.ncdc.noaa.gov/sites/default/files/Integrated-Surface-Database-Stations-Over-Time-Chart.jpg)

The dataset is **awesome**, and huge. It's also shipped in a pretty disgusting format that needs an [enormous Java class with lots of horrible static members](http://www1.ncdc.noaa.gov/pub/data/noaa/ishJava.java) to let you parse it into something meaningful.

You can browse the dataset [HERE](http://www1.ncdc.noaa.gov/pub/data/noaa/). The (infrequently updated) 'index' of weather stations is also available at that location under the name **ish-history.csv**. 

Because all of the data for the stations is stored in individually g-zipped files per station, per year, this dataset can be a pain to download. You end up having to grab thousands of small files for which the overhead of making the http connections each time slows things down. ACK, indeed!

![ACK! - originally from http://www.centoscn.com/uploads/allimg/140306/00564AM6-0.jpg](/images/ack.jpg)

I eventually hacked some [messy scripts](https://github.com/SimonLeigh/couchbaseNoaaLoader) together to parallelise this effort. They can be found over at my GitHub.

OK, so we've messed around with the best way to crawl the NOAA website and download the files. Now we have tens of gigabytes of compressed data and we want to store it in Couchbase and use it to power an application.

I like to map backwards from application requirements I need to the Couchbase documents. I am going to cheat here to avoid drawing a diagram and use the front-end to highlight the things I considered!

---

![NOAA Data Frontend](/images/noaa-frontend.png)

1. _Each weather station needs to be individually identifiable and retrievable, and we need to be able to search for them by their location._
2. _We need to be able to retrieve data down to a daily granularity._
3. _We want to be able to aggregate and filter the measurements over the day into a summary box._
4. _We want to be able to retrieve the data in a lightning fast manner and plot it on a chart (the chart is off the screen in the image above)._

---


### Weather Stations (1)

The most natural mapping for the weather stations was to just store them in one document each. When it came to designing keys, I just used a concatenation of two fields that together comprise a unique identifier for the station. These documents look like this:

```javascript

var key = "station::012270-99999"
var value = {
	"WBAN":"99999",
	"USAF":"012270",
	"STATION NAME":"INNERDALEN",
	"CTRY":"NO","END":"20150831",
	"LON":"+008.783",
	"ICAO":"ENKB",
	"BEGIN":"19730101",
	"type":"station",
	"LAT":"+62.717",
	"ELEV(M)":"+0405.0"
	}
```

### Sample Measurements (2,3,4)

Since I knew that I would always access all the elements for a single day, I decided to aggregate an entire day's measurements for a station into one document per day. This still gave me access to all the measurements but I had to retrieve the whole day's worth in one go. Since Couchbase can store and index JSON, it was easy to pack an array of sub-documents into a top level JSON document. The beauty of JSON meant that I didn't care whether there were 2 or 2000 samples in a day - my array would just be a different length. 

I decided to put the date of the measurement in the key - this way, all I need to know is the ID of the weather station and the date I want and I can use a **key-value** access to retrieve the documents with sub-millisecond response times. I did not have to traverse a secondary index.

These documents look like this:

```javascript

var key = "722362-93937::2015-08-10"
var value = {
    "samples": [
    {
        "TEMP": "99",
        "VSB": "10.0",
        "DEWP": "56",
        "SPD": "6",
        "ALT": "29.90",
        "TIME": "0015",
        "SKC": "CLR",
        "DIR": "160",
        "STP": "994.8",
        "CLG": "722"
    },
     ...
    {
        "TEMP": "103",
        "MIN": "100",
        "VSB": "10.0",
        "DEWP": "56",
        "MAX": "104",
        "SPD": "3",
        "ALT": "29.88",
        "TIME": "2355",
        "SKC": "CLR",
        "DIR": "020",
        "STP": "994.1",
        "CLG": "722"
    }
    ]
}

```

Astute readers may notice that I stored the values as strings, even when they were floats or integers. There's no good reason for this, and in fact it was a result of using the NOAA provided Java class. A future iteration of the parser should store these values as numeric types.

So, now we've decided on our data model and we can load the data into Couchbase. This dataset is actually well over 100GB in size for the entire ~100 years.

In my next blog I'll dig into how this model lets us create a fast, responsive web application that will let us search for and query the weather anywhere on the planet and on any day for the last hundred years!

