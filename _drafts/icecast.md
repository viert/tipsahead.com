---
layout: post
title: "Making of online MP3 radio: 1. Icecast"
date: 2014-05-21 12:00:00
categories: radio
cut: 1055
---

If you've never visited [viert.fm], maybe it's time. Once I've found myself listening to radiostation and thinking "Man,
I always wanted to be a DJ!" 

Well I'm just kidding. Actually once I've decided to try making live online-radio show. I won't speak about client-side
application for broadcasting, it's a big deal and not a part of a tech blog. But the main idea is that I've decided to
make an online radio to broadcast at least shuffled playlist. Then the idea grew bigger: shuffle must be smart enough
not to play the same artist in a row, there must be the queue to allow users to order their favorite songs. And users
must have limits not allowing them to order too much. And there must be a web-application. And so on, and so on...

So here I want to tell you how to create this kind of app from scratch. It's rather easy and could be fun. [viert.fm] 
is written using Ruby-on-Rails but here I've decided to experiment with [sails.js]. 

But anyway the first step is installing the broadcasting software. The only one seemed adequate was [Icecast]. Thanks
God it's cool enough to stop suffering from lack of alternatives. We don't need much from it: it must, well, broadcast
 audiostream, it must authorize users without http basic authorization (because browser auth window sucks and we're 
 making beautiful modern web app) and it must have flexible console client so we can easily manipulate song order. 
 
[Icecast] has it all, so let's get started.





[viert.fm]: http://viert.fm
[sails.js]: http://sailsjs.org
[Icecast]: http://www.icecast.org