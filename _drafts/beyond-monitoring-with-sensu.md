---
layout: post
draft: true
title: Beyond System Monitoring with Sensu
date: '2014-02-17'
description: Sensu is great at system monitoring and alerting, but you can also send application alerts and metrics through it.
categories: ['monitoring']
versions:
    - name: Sensu
      version: '1.xx'

---

## Sensu's Architecture

Sensu has `server` and `client`, components with the client running scripts and sending results back to the `server` for handling. Communication between the server and client is handled over AMQP - specifically, RabbitMQ.

With Sensu, there are both `Metrics` and `Checks`. Metrics are key-value pairs of a name and integer value, used for things like memory, cpu usage, and whatever else you may normally throw up on a chart. Checks are boolean - a service is up or down; a disk is 80% full (or more) or not; the network is saturated or not, and so on. Both checks and metrics define a list of `Handlers` that should process their results.

There are two ways for Sensu to get these requests, pushes and pulls. With pulls, the `server` handles the scheduling, and sends out requests for metrics or checks to be run over AMQP Channels. The clients, which listen to different channels it subscribes to, then runs the requested script and sends back the data. With pushes, the client runs the checks or metrics on it's own schedules, and sends the data over the AMQP Chennel.

### Handlers

When it receives a result from a Client, the Sensu Server can send payload data over TCP, UDP, or AMQP, or it can pipe data into other scripts. Handlers can also be grouped together, so that multiple Handlers are run for each result. 



### blah blah blah

By using the alerting mechanisms that you've already setup in Sensu, we can write a single wrapper library to use in all of our services to handle exceptions. Then, by using different Sensu Handlers, we can vary where alerts or metrics are sent to.

For instance, 

