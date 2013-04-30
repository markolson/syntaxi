---
layout: post
title: So Why Can't Twitter Keep Track of My Direct Messages?
date: '2013-04-20'
description:
categories: ['api']
---
<blockquote class="twitter-tweet" data-conversation="none"><p>Remembering that a user has read a Twitter direct message is the hardest problem in computer science.</p>&mdash; Robin Ward (@evil_trout) <a href="https://twitter.com/evil_trout/status/317528358751711232">March 29, 2013</a></blockquote>

I get annoyed when I read a Twitter Direct Message in my iPhone app, and the next time I open the website, it says I have a new Direct Message. And then when I open a Twitter app on another device, it's still unread. I have to read the message over and over, even if I replied to it. How can it be 2013 and this is allowed to be the case? <strong><em>How?</em></strong>

The pre-emptive TL;DR for this is because the way Twitter gives apps (and the website) data through its API only care about what's happening right now, and incidentally, what happened just before right now. There's just no method to say "this is the last thing I read" in Twitter API-ese, and the way that Direct Messages are handled make it even more complex for apps to deal with.

<!-- more -->

## The Twitter API

Twitter's <a href="https://dev.twitter.com/docs">Dev site</a> is pretty good throughout, but there's just a bit from the <a href="https://dev.twitter.com/docs/api/1.1">API Documentation itself</a> needed to explain this. Quoting those docs about the home timeline, the one that normally shows up when you open the website or an app:

<blockquote>Returns a collection of the most recent Tweets and retweets posted by the authenticating user and the users they follow.</blockquote>

The important bit is that it says <em>"the most recent"</em>, not <em>"the most recent since a user last accessed the timeline"</em> or something similar. There's nothing built into the API that cares that you haven't accessed this feed for thirty weeks and have missed 290,142 tweets. While you can <a href="https://dev.twitter.com/docs/working-with-timelines">request and get back older tweets</a> from any collection, by default it's just the newest. <strong>There is a small, important wrinkle with how DMs are handled, though. They're not seperated out per-conversation:</strong>

<blockquote>Returns the 20 most recent direct messages sent to the authenticating user. Includes detailed information about the sender and recipient user.</blockquote>

In a stripped down way, that means your Twitter app gets a list of messages that look like:
{%highlight javascript %}
	[
		{id: 345, from: "mark_olson", to: "a_friend", message: "you'll be there tonight, right?"},
		{id: 541, from: "a_friend", to: "mark_olson", message: "yep."},
		{id: 546, from: "someone_else", to: "mark_olson", message: "they're answering DMs, but not texts again?"},
		{id: 547, from: "a_friend", to: "mark_olson", message: "at 8, right?"},
	]
{% endhighlight %}
It's up to the app itself to recognize that there are two conversations going on, and to show them as two seperate conversation with two seperate "unread" notifications. Internally it can know that I've seen the DM with ID #547 and am up to date on that conversation (changing the UI accordingly), but it can't assume I've seen #546 - even with its lower ID - because it's a conversation with a different person. More importantly, there's no way that the app can tell Twitter that its seen either conversation at all.

## A Partway Solution: Add More API

The workaround for keeping track of the last tweet that a user was shown is to use another service which is accessable from all the devices the app runs on. For <a href="http://twitter.com/qwantz">@qwantz</a> to keep track of the last tweet it read, I use <a href="http://tweetmarker.net">Tweet Marker</a>, a bare bones service that only exists to keep track of that information. The basic usage is to `POST` a bit of data containing the Twitter collection name (eg, `home`, `mentions`, `retweets`) and the last ID seen to its API. For example:

{%highlight javascript %}
	{  "home":  {  "id": "123456"  }  }
{% endhighlight %}

You can then request that bit of data from any authorized app, and have them all know that you just read tweet "123456" on my "home" timeline, and to skip ahead to that. As you can see on the Tweet Marker page, a lot of apps use this service. Because your app may already keep some of your timelines (particularly the @ mentions one, which also highlights a "unread" UI element in many apps) in sync without you noticing it, the lack of synced DMs can be a bit jarring and irksome.

To be honest, I don't particularly know why the apps I use don't sync DMs, because I know they use Tweet Marker and all I think it would require is adding a key like:

{%highlight javascript %}
	{  "dm.a_friend":  {  "id": "547"  }  }
{% endhighlight %}

Then in each app, upon seeing that there's a DM conversation with `a_friend`, it could check what the value for `dm.a_friend` was, compare it with the largest ID in the conversation for that person, and only then change the UI to say if there was a new message if needed. That'd solve unsync'd DMs, at least across apps that use Tweet Marker.

I think.
