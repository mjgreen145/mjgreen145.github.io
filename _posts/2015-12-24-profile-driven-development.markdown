---
layout: post
title:  "Profile Driven Development"
date:   2015-12-24 20:11:00 +0000
---

Now, don't worry. I'm not here to force a new programming methodoligy upon you. Instead I want to highlight how profiling can, and should, be an important part of your development process, and how it can help produce quality code.

<!--more-->

A few months ago, we (the tech team for Net-A-Porter.com) were getting ready to launch our newest application - a new webapp to serve all of our product pages. But during load testing, we noticed very high levels of CPU on the server.

As far as the server is concerned, it's a fairly simple app. Written in Node.js, it makes HTTP requests to a couple of our internal APIs, and then spits out some HTML. Therefore high CPU levels seemed odd for a process which doesn't do much in the way of CPU-intensive work. Some investigation needed to occur.

### Introducing the V8-profiler

I'm not overly familiar with profiling tools, but after a small bit of internet-ing I came across the [v8-profiler](https://www.npmjs.com/package/v8-profiler). We use Express for our web framework, so I started profiling at the entry point of a request:

{% highlight javascript %}
var profiler = require('v8-profiler');
app.get('/:countryIso/:langIso/product/:pid',
    function(req, res, next) {
        profiler.startProfiling('product-page');
        next();
    },
    // more middleware
);
{% endhighlight %}

Then, I stop profiling just before sending the payload back to the user, and save the profile data to disk:

{% highlight javascript %}
var cpuProfile = profiler.stopProfiling('product-page');
cpuProfile.export(function(error, result) {
    fs.writeFileSync('profile.cpuprofile', result);
    cpuProfile.delete();
});
{% endhighlight %}

By using the extension .cpuprofile, the saved data can be loaded into Chrome's profiles panel in DevTools, giving a nice graphic of what's going on:

![Load profiles into Chrome for useful charts](/static/profile-driven-development/profile_before.png)
*Load profiles into Chrome for useful charts*

### Spotting the bottleneck

This may look very confusing at first. Express uses the middleware pattern, meaning that the call stack gets deeper as you go through the chain of middleware. Each time you see 'next' followed by 'handle', it means Express is going to the next piece of middleware. The way to then spot a bottleneck in this chart is to look for two instances of 'next' being called with a large amount of time between them.

In this case, a lot of time is being spent doing some sort of compiling:

![Visualising the data makes problems easy to spot](/static/profile-driven-development/profile_before_highlighted.png)
*Visualising the data makes problems easy to spot*

Looking at the code, I realise that we're using [Handlebars](http://handlebarsjs.com/) to convert template URLs into actual URLs. But the real crippling factor, is that we're re-compiling the template string _every time_ we generate a URL. Compilation of a template string into a compiled temaplte is slow - intentionally so. By doing most of the work up-front, the runtime action of turning a compiled template into a populated string can be made as fast as possible.

With this information uncovered, a fix is easy. We _could_ compile the template string and save it for re-use. But, the template is slightly different for each product on the website, so we'd have to do at least one compilation per request. A better alternative, which avoids the use of a library entirely, is to simply use `String.replace()` to replace each variable in the string for the value we want.

### The Results

Running the improved code through the V8-profiler again yeilds this profile chart:

![Ahh, much better](/static/profile-driven-development/profile_after_highlighted.png)
*Ahh, much better*

The bits highlighted in the red box are the pieces of middleware which were previously taking a long time. Now they are quick enough for the profile to consider them taking zero time.

The impact on CPU was clear as well. When put through another load test, rather than CPU being pegged at 95-100% for the duration, it increased and decreased with the amount of load being put through it. This is exactly what we'd expect to see from a system such as this.

Whilst this bit of code doesn't need optimising any further yet, this did get me thinking - is `String.replace()` the best method for this task? For the answer to that, you have to wait until next time...
