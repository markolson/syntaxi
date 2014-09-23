---
layout: post
title: Setting Up Carbon Relay For Graphite
date: '2014-03-01'
description: I think that Carbon, Whisper, and Graphite is *delightfully* under-documented, which makes figuring out how to do something simple like send metrics to 2 hosts ... fun.
categories: ['monitoring']
newb: true
versions:
    - name: Graphite
      version: '0.9.12'
---

As part of a recent project at work, we've wanted to take metrics that different systems (each made up of multipe nodes) gather and use locally to a central store where we can monitor and compare different systems. I knew _of_ `graphite-relay` and I had a notional idea of how it worked, but because of Graphite's charmingly incomplete documentation, I wasn't sure how to actually set it up. 

The key to my understanding of getting `carbon-relay` working was realizing that `carbon-relay` transmits pickled data, not raw metrics. Given a local `carbon` instance on `localhost`, and a remote one that we want to copy the results to at `stat-sink.corp`, you'd start by modifying `carbon-relay.conf` and setting the destinations.

```
[default]
default = true
destinations = stat-sink.corp:2014, 127.0.0.1:2014
```

Because we're sending the metrics to `carbon-relay` instead of `carbon`, we have to have `carbon-relay` listen on the port that our metric collectors are sending data to. To do this, we edit entries `carbon.conf`, adding the `DESTINATIONS` entries from above, and switching the `LINE_RECEIVER_PORT` & `PICKLE_RECEIVER_PORT` entries from `[cache]` and `[relay]` sections. Omitting many, many lines: 

```
[cache]
LINE_RECEIVER_PORT = 2003
PICKLE_RECEIVER_PORT = 2004
[relay]
LINE_RECEIVER_PORT = 2013
PICKLE_RECEIVER_PORT = 2014
```

Should be the following:

```
[cache]
LINE_RECEIVER_PORT = 2013
PICKLE_RECEIVER_PORT = 2014
[relay]
DESTINATIONS = stat-sink.corp:2014, 127.0.0.1:2014
LINE_RECEIVER_PORT = 2003
PICKLE_RECEIVER_PORT = 2004
```

After the ports are changed and destinations added, restart `carbon` and start `carbon-relay`. Both localhost and stat-sink should begin recieving any metrics that are sent to the relay now. Hooray! With dozens of systems sending metrics to stat-sink, we can plot multiple at once and immediately see any outliers. 

I know that Carbon and Whisper setups can get very involved, with all sorts of options around storage load balancing, but this simple 1:1 duplication turned out to be a nice way to dip our toes into the larger sea of monitoring at a bigger scale.
