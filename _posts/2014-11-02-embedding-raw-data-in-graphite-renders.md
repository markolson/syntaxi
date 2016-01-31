---
layout: post
title: "Embedding Raw Data in Graphite Renders"
date: '2014-08-22'
description:
categories: ['monitoring']
newb: true
versions:
    - name: Graphite
      version: '0.9.13-pre'
---

<img src="/assets/media/graphite-embedding/render.png" />

How you take snapshots of single metric charts, or, entire dashboards, is an interesting problem to me. Because the raw metrics are eventually rolled up into larger and larger aggregations before being dropped entirely, if something 'interesting' happens that you'll want to reference for the future, you have to export the raw data - the image just won't do. While the image can be reconstructed using the raw data, the opposite isn't true. I can squint at the image above trying to figure out what the current was at 13:30, but that's a fools errand.

As it happens, it's not terribly difficult to get Graphite to store the raw metrics along with the rendered image. The above image was rendered by Graphite with <a href="https://gist.github.com/markolson/ecd6e46a9bea9878b3d1#file-embed-patch">this small patch</a> applied to it. It's also pretty simple to make a <a href="https://gist.github.com/markolson/ecd6e46a9bea9878b3d1#file-parser-rb">simple PNG parser</a> that can pull out data in `tEXt` chunks, getting the metric name and timeseries points. Of course, running `strings` on the file can get you that as well, since it's stored in plain text.

I have no idea if this is actually useful, but there's something appealing to me about being able to more easily keep raw data with visualizations. I doubt that I'll clean this up and make it into an actual PR for Graphite - the dependency on PIL is a bit much for such a small thing, and even though PNG is simple enough that you could drop the right bytes in with some string manipulation, that feels too dirty to actually do. Still, I like this idea.
