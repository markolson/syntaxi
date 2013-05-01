---
layout: post
title: Let's Explore Kickstarter's API
date: '2013-03-24'
description: Kickstarter hasn't documented their API, but their iOS app uses one...
categories: ['api']
---

I like to save as many emails, pictures, chat logs, and other digital ephemera of my life as I can. I like that, more than just seeing the pictures from a skydiving trip, I have all the emails, facebook statuses, and tweets that went along with the event - it makes the story richer.

As part of this hoarding, I  decided I wanted a details about all of Kickstarter projects I backed, and went looking for an API I could use to pull the data down. Googling and digging through the site, all the data I could find were <a href="http://www.kickstarter.com/help/stats">a simple year-on-year stats page</a> and some <a href="http://www.kickstarter.com/blog/categories/data">blog posts</a> about milestones the service has hit. As it happens, Kickstarter does have an API - they just haven't opened or documented it up yet.

## Finding the API

Because it seemed like a safe place to start, I opened https://api.kickstarter.com, assuming that there would be a registration page, or maybe some docs. Alas, all I got was

```javascript
	{"error_messages":["You are not authorized to access this resource."],"http_code":401,"ksr_code":"unauthorized"}
```

This is quite a bit better than nothing, as it tells me that there are resources I could potentially be authorized to access. <!-- more -->And Kickstarter did just release <a href="http://www.kickstarter.com/blog/introducing-the-kickstarter-app-for-iphone-and-ipo">their iPhone app</a> a month earlier, and that let me sign in, so that's a good place to start. In the past, I've used <a href="http://www.charlesproxy.com">Charles</a>, a "HTTP proxy / HTTP monitor / Reverse Proxy" to peek at traffic going between my phone and the internet, so I set it up once again and got to playing around with the app.

Kickstarter's API has at least two authentication paths that return an `oauth_token`: one uses facebook's oauth, and the other uses some login credentials. Authenticating (with some patently fake credentials) is easy:

```bash
	curl -X POST -d '{"password":"not-a-real-password","email":"not-a-real-email@example.com"}' https://api.kickstarter.com/xauth/access_token?client_id=2II5GGBZLOOZAA5XBU1U0Y44BU57Q58L8KOGM7H0E0YFHP3KTG
```

If successful, a JSON blob with top level keys of an `access_token` string and `user` object is returned:

```javascript
{
	"access_token":"410not_a_real_access_token",
	"user": {
		"name":"First Last",
		"slug":"username",
		"avatar":{...}
		...
	}
}
```

Using that `access_token` as a query paramter, I could GET the API's root and see that Kickstarter has already gone to some measure of trouble creating a bit of a hypermedia API

```bash
curl https://api.kickstarter.com/v1/?oauth_token=410not_a_real_access_token
```

```javascript
{
	"urls": {
		"api": {
			"categories": "https://api.kickstarter.com/v1/categories",
			"newest_projects": "https://api.kickstarter.com/v1/projects/newest",
			"popular_projects": "https://api.kickstarter.com/v1/projects/popular",
			"ending_soon_projects": "https://api.kickstarter.com/v1/projects/ending_soon",
			"near_projects": "https://api.kickstarter.com/v1/projects/near",
			"search_projects": "https://api.kickstarter.com/v1/projects/search",
			"self" :"https://api.kickstarter.com/v1/users/self"
			}
		}
}
```

In fact, each response from the API contains a `urls` object, that itself has `api` and `web` objects with references to other API endpoints. Thanks to this self documenting aspect of the API, where you can just look at what's in object.urls.api, it was pretty easy to build up a little Ruby Gem: <a href="https://github.com/markolson/kickscraper">kickscraper</a>.

## <a href="https://github.com/markolson/kickscraper">Kickscraper</a>

I admittedly probably spent too much time on kickscraper based on what I wanted out of it - a list of projects I backed, the reward level and date of that backing, and if the project succeeded - but heck, going overboard can be fun for it's own sake. Turns out the API doesn't return any data about what level you backed a project at (<a href="#mobile">they open a webpage in the app for that info</a>), but I could at least get a list of projects I backed, along with if they met their goal or not. Combined with the emails I get from Kickstarter, this is *Good Enough* for my collect-everything project, though I definitely hope to improve on it as more data becomes available. Here's a quick sample of what kickscraper can do:

```ruby
require 'kickscraper'

Kickscraper.configure do |config|
        config.email = 'not-a-real-email@example.com'
        config.password = 'not-a-real-password'
end

c = Kickscraper.client
puts " Active | Met Goal"
puts "------------------"
c.user.backed_projects.each {|x|
        print (x.active? ? '    X   |' : '        |')
        print (x.successful? ? '    X     | ' : '          | ')
        puts x.name
        # x.rewards # if you want a list of all the reward levels.
}
```

Running that will give this overview, though the script I used dumped my data into a little CSV.

```text
 Active | Met Goal
--------------------
    X   |          | Homeworld Touch (iOS/Android) and Homeworld 3 (PC/Mac/Linux)
    X   |    X     | The Veronica Mars Movie Project
    X   |    X     | RiffTrax Wants to Riff TWILIGHT Live in Theaters Nationwide!
        |    X     | To Be Or Not To Be: That Is The Adventure
        |    X     | Planetary Annihilation - A Next Generation RTS
        |    X     | OUYA: A New Kind of Video Game Console
        |    X     | Gotham Knight Terrors: Comedic Batman Short
        |          | Internet Meme Playing Cards
        |    X     | GOOD JOB, BRAIN! - A Trivia & Quiz Show Podcast
        |    X     | Elevation Dock: The Best Dock For iPhone
```

There are more methods and quick samples documented in the <a href="https://github.com/markolson/kickscraper#readme">README</a>, and a more comprehensive list of what data is accessable on a <a href="https://github.com/markolson/kickscraper/wiki/Datatypes">wiki page</a> and while it can still use a lot of work - mostly with error handling and paging - I think it's pretty usable. 

## Bonus: A Mobile Site

One thing that I did notice is that a few areas of the app aren't generated from API data - they're just webviews. Adding `Kickstarter-iOS-app: 339` to the request header will return a different project page, and perhaps more.
