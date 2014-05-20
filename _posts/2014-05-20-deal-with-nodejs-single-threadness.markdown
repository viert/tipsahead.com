---
layout: post
title:  "Deal with node.js single-threadness"
date:   2014-05-20 11:37:06
categories: nodejs
summary: "For the last 4 years I coded all my projects using Ruby on Rails. It suited me completely until I faced 
          really high loads. Since then I began to search for alternatives. Everything I tried was not at least 10% as comfortable,
          intuitive and elegant as RoR for me, so I kept on trying to speed up ruby using new faster language versions, rails cache 
          system, event-based servers and so forth. Still I consider Rails to be the best choise for fast bootstrap of nearly any project."
---

For the last 4 years I coded all my projects using Ruby on Rails. It suited me completely until I faced 
really high loads. Since then I began to search for alternatives. Everything I tried was not at least 10% as comfortable,
intuitive and elegant as RoR for me, so I kept on trying to speed up ruby using new faster language versions, rails cache 
system, event-based servers and so forth. Still I consider Rails to be the best choise for fast bootstrap of nearly any
project.

Ok, a few days ago I've found [sails.js][sails] framework. It reminded me Rails much more than anything else among web 
frameworks that I know. To tell the truth there are [Grails] besides but I really don't like JVM and even can't explain to myself why. 
My familiar Java-coder says it's ok, I'm just that kind of man :)

So I've found [sails.js][sails] and tried to use it. It's really ok, every web programmer one way or another face with
JavaScript so I gave node.js a chance. One simple synthetic benchmark convinced me it's already kind of cool thing.

**Ruby code**
{% highlight ruby %}
#!/usr/bin/env ruby

def a
  i = 0
  (100000001).times do |j|
    i += j
  end
  return i
end

t1 = Time.now
puts a
t2 = Time.now
puts "#{t2-t1} seconds"
{% endhighlight %}

**Javascript code**
{% highlight javascript %}
a = function() {
  var i = 0;
  for (var j=0; j<100000001; j++) {
    i += j;
  }
  return i;
}

t1 = (new Date()).getTime();
console.log(a());
t2 = (new Date()).getTime();
console.log((t2 - t1)/1000 + " seconds");
{% endhighlight %}

This two code examples seem identical though written in different languages. And these are the results.

{% highlight text %}
viert@aqua:~/src/bench$ ruby bench.rb 
5000000050000000
5.912625616 seconds

viert@aqua:~/src/bench$ nodejs bench.js 
5000000050000000
0.148 seconds
{% endhighlight %}

Simple arithmetic operations in node.js are 40 times faster. Also javascript interpreters are optimised to work with
 strings concatenation so templating must be pretty easy task for node.js. JSON parsing is a part of language itself
 so it has to be fast also.
  
One thing you can't get from JavaScript is multithreaded parallel execution of your code. Every node.js tutorial says 
_"Everything in node.js runs in parallel except your own code"_. What does it mean? Let's take we have simple sails.js 
application with two handlers: `/fast` and `/slow`. In your controller it may look like this:

{% highlight javascript %}
var sleep = require('sleep');

module.exports = {
    fast: function(req, res) {
        res.end('Fast Ok');
    },
    slow: function(req, res) {
        sleep.sleep(10);
        res.end('Slow Ok');
    }
}
{% endhighlight %}

If you run the app, get `/slow` handler from your browser and right away get `/fast` handler from another browser's tab,
 you'll see that fast handler works for the same 10 seconds i.e. it completes not before the completion of the slow 
 handler. That's it, node.js is single-threaded and **your code** always runs serially. It means node.js keeps on accepting
 connections but it can't run your handler in time so it just put this task in the queue for javascript event-loop. 
 
What can you do? The most right answer is _"You should avoid heavy calculations in your application"_. The most right 
 way is to use node.js as nothing more than template engine. It can handle user connections well, it also can get data
 from database/file system/http backend/whatever without blocking your code because IO in nodejs is asynchronous and runs
 separately from your code. Best practice for node.js is to be on the top of 3-level app architecture: node.js is frontend,
 then backend follows written in fast language with multithreading support, and the data store underneath.
 
But what if we have no choice? If you ever used [Graphite] you know it can render graphs from hundred metrics aggregating
them on-the-fly. Let's suppose we're making that kind of handler. Getting hundred metrics may be slow, but node.js do this
work asynchronously without blocking the entire application. But then we have to reduce this results i.e. to get **sum** of
them and then to give the results back to user. Let's emulate these heavy calculations replacing it with the syntetic benchmark test
we discussed above:

{% highlight javascript %}
module.exports = {
  fast: function(req, res) {
    res.end('Fast Ok');
  },
  slow: function(req, res) {
    var j = 0;
    for (var i=0; i < 4000000000; i++) {
      j += i;
    }
    res.end('Slow OK: ' + j);
  }
};
{% endhighlight %}

In this example `/slow` handler still blocks app for a few seconds and then returns `Slow OK: 7999999996067115000`. 
Now all we have to do is to split handler to microtasks, each followed by returning control to javascript event loop.

{% highlight javascript %}
module.exports = {
  fast: function(req, res) {
    res.end('Fast Ok');
  },
  slow: function(req, res) {
    var step = 100000;
    var task = function(acc, start, end) {
      if (start == end) {
        return res.end('Slow OK: ' + acc);
      }
      for (var i=start; i<start+step; i++) {
        acc += i;
      }
      setImmediate(task, acc, start+step, end);
    }

    task(0, 0, 4000000000);
  }
};
{% endhighlight %}

This code blocks app for a couple of milliseconds while calculating a small portion of the whole big task. You can test the `/fast`
 handler to be sure it's working pretty fast now. What happened? We just run task that consists of the portion of loops, to be more
 precise, `step` loops at a time. Then we call setImmediate that just returns control to event loop and puts the next
 iteration of our task to the event-loop queue. If there are other callbacks in queue, they can run fast and then interpreter
 proceeds to the next iteration of the `slow` task.
  
Problem seems to be resolved but the resolution looks a bit complicated so I still recommend to avoid heavy calculations in Javascript.


[Graphite]: http://graphite.wikidot.com/
[Grails]: http://grails.org
[sails]: http://sailsjs.org
