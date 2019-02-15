---
layout: post
title: "WebSockets in Ruby"
date: 2014-02-26 09:03
comments: true
categories: blog
tags: [ruby, websockets, rails, web, celluloid, webmachine-ruby]
---

At my main job, we have a large datastructure that takes considerable CPU time to be built, but remains unchanged thereafter. Its job is to geocode positions to and from a local reference system, which in turn provides us the ability to pin records, for instance, to a place on a Road, and know to which coordinate pair a local reference would correspond.

For the first pass, I built the Ruby library for the geocoding and a simple (Sinatra-based) webservice. This worked fine for a while until the Client required that **every** mouse move performed a conversion. Said change, prompted me to build the same geocoding infrastructure again in JavaScript, and all were happy for a while.

As it usually goes, a new decision was made to support multiple Roads per User. Now, a download of 800KB of data (stored in an IndexedDB for later sessions) was tolerable; potentially multiple megabytes would be deadly, even if the software could be used before that constant feedback of conversions was given &mdash; it just became one of those features Users hold on to.

I knew that we had to go for a solution that kept that intact and made the whole thing manageable. I had dabbled in WebSockets before (with [node.js][nodejs] and [Socket.IO][socketio]) and kind of knew the lay of the land. Still, from previous searches, I was also aware there was a dearth of Ruby solutions, and for a moment considered going with my JavaScript port on node. The thought gave me shivers.

## The contenders

The first step was finding out what could be used. This is what I evaluated:

  * [sinatra-websocket][sinatraws]
  * [faye-websocket][fayews]
  * [websocket-rails][wsrails]
  * [tubesock][tubesock]
  * [webmachine-ruby][webmachine]

The first three are EventMachine-based; _tubesock_ uses [rack hijacking][rackhj]; _webmachine-ruby_ provides WebSockets via [Reel][reel], a Celluloid::IO-based HTTP server.

At first, considering I was already using Sinatra, I tried _sinatra-websocket_. For some reason I just couldn't get the connection to be upgraded to a WebSocket, and decided to move on quickly. _faye-websocket_ I just skipped, to be frank.

The next two suffered from the same problem: after booting Rails and loading the structure, I was left with only enough memory for a couple dozen or so clients on a small Heroku dyno. Also, Rails' boot time coupled with building the thing occasionally made Heroku think something had gone wrong, and often the process crashed before the service went up.

The only one left, if you're counting, was _webmachine-ruby_.

## webmachine-ruby

Setting up was relatively easy. To ramp up, I first migrated the original HTTP-based service to its resource structure. It has more of an OO flair than both Rails and Sinatra, with the caveat that it provides a lot less (by design). The dispatcher is easy to understand, and I quite enjoyed toying with the [visual debugger][visualdbwm].

Moving to a WebSocket, however, changes everything. As far as I can tell (and the documentation specifies) you completely skip over the regular infrastructure by providing a _callable_ to a configuration option, as such:

```
App = Webmachine::Application do |app|
  app.configure do |config|
    config.adapter = :Reel
    config.adapter_options[:websocket_handler] = proc do |websocket|
      websocket << "hello, world"
    end
  end
end
```

That is pretty much what the docs say. Since it only expects the handler to respond to _#call_, you can write your own _ad-hoc_ dispatcher:

```
class WebsocketHandler
  def call(websocket)
    message = websocket.read
    # do something with the message, call methods on other objects, log stuff, have your fun
  end
end
```

What the docs don't address are some basics of sockets programming. If you see your handler hang and never respond again, requiring you to restart, don't fret: you just have to provide a loop to read from the the socket and let Celluloid::IO do its non-blocking magic:

```
class WebsocketHandler
  def call(websocket)
    loop do
      message = websocket.read
      # do something with the message, call methods on other objects, log stuff, have your fun
    end
  end
end
```

Don't worry: your CPU won't be pegged at 100%, because non-blocking. You'll be subjected, however, to the same limitations node has regarding CPU usage and its event handlers (i.e. if you are CPU-intensive, you'll affect throughput).

Luckily, we have threads in Ruby. I decided to take advantage of that by assigning each client to a Celluloid Actor, which allows me to provide some of the CPU-intensive operations without compromising (at least not heavily) other Users. It has been working fine so far.

## What's missing

My solution doesn't take into account non-WebSocket clients, but it should. _webmachine-ruby_ makes it easy by allowing you to implement streaming APIs without much trouble, and I suppose it'll only take a bit of JS to fallback from one to the other and provide an abstract connection to consumers.

The documentation also doesn't go over all the events that can happen on the socket (_onerror_, _onclose_, _onopen_, _onmessage_). You can see them as methods on the socket, each taking a block, but for my use case I just let the actor crash and be done with it. If I'm missing some cleanup, please let me know.

This architecture also doesn't provide a ready-baked pub/sub system, with channels and message brokers. If that's more in the spirit of what you need, check out [faye][faye] and [websocket-rails][wsrails].

[faye]: http://faye.jcoglan.com/
[fayews]: https://github.com/faye/faye-websocket-ruby
[socketio]: http://socket.io/
[nodejs]: http://nodejs.org
[sinatraws]: https://github.com/simulacre/sinatra-websocket
[tubesock]: https://github.com/ngauthier/tubesock
[wsrails]: https://github.com/websocket-rails/websocket-rails
[webmachine]: https://github.com/seancribbs/webmachine-ruby
[rackhj]: http://blog.phusion.nl/2013/01/23/the-new-rack-socket-hijacking-api/
[reel]: https://github.com/celluloid/reel
[visualdbwm]: https://github.com/seancribbs/webmachine-ruby#visual-debugger
