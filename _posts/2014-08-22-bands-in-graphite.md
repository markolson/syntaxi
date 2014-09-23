---
layout: post
title: "Creating Error Bands with Graphite"
date: '2014-08-22'
description: Get some 'simple' caution, danger, or error bands with Graphite for when you want to see your thresholds as more than just a single line.
categories: ['monitoring']
newb: true
versions:
    - name: Graphite
      version: '0.9.13-pre'
---

In the (long) upcoming [0.9.3 release](https://github.com/graphite-project/graphite-web/blob/master/docs/releases/0_9_13.rst) of Graphite, I made a quick patch for a small problem that I'd been itching for fix for a while - adding error bands on graphs. While I love having my email and chat slammed with threshold alerts, I've also thought it would be nice to see a historical perspective of those thresholds in the charts. The end solution is decidedly *not* pretty, but it's functional and gives me something useful to look at.

To start, I'd have a normal chart showing something getting measured. I'd separately have threshold alerts on this, either collecting the metric independently or pulling json data from graphite.

     render?target=alias(metric.name, 'data')&yMin=0&yMax=100

<img src="/assets/media/graphite-error-bands/render-1.png" />

What the triggered alerts didn't easily show me was if there was any pattern to the metric crossing that threshold - was it correlated with another metric? a time of day? They could show me timestamps of a few past events, but that was it. With `constantLine`, I could overlay a simple line at a y-value that lined up with the 'danger' and 'critical' thresholds the alerts were sent out at.

Zooming out on a metric, I could eyeball patterns, or add metrics targets to be rendered.

    render?target=alias(metric.name, 'data')&yMin=0&yMax=100
    &target=alias(color(constantLine(75), 'yellow'), 'danger')
    &target=alias(color(constantLine(90), 'red'), 'error')

<img src="/assets/media/graphite-error-bands/render-2.png" />

That works, but dang if that yellow isn't hard to see. Both lines can also be hard to make out of there are lots metrics in the graph, which reduces their usefulness. Graphite has `areaBetween`, which lets you set a background color (and label) that goes between 2 other metrics, which sounded perfect. 

    render?target=alias(metric.name, 'data')&yMin=0&yMax=100
    &target=alpha(color(alias(areaBetween(group(constantLine(70),constantLine(90))),""),"yellow"),0.5)
    &target=alpha(color(alias(areaBetween(group(constantLine(90),constantLine(100))),""),"red"),0.5)

<img src="/assets/media/graphite-error-bands/render-3.png" />

Before 0.9.13, trying to use `areaBetween` with `constantLine`s failed with a bit of a cryptic (to an end-user) error. Eventually I dug in, [made a patch](https://github.com/graphite-project/graphite-web/pull/897/files) and happily got it accepted, letting me make my silly graphs. I used a nameless `alias` in this because to me, the meaning of yellow and red backgrounds is known, whereas the `constantLine`s alone could be misinterpreted as metrics.