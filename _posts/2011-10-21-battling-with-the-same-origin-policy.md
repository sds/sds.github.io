---
layout: post
title: Battling with the Same Origin Policy
tags:
- hack
- HTTP
- HTTPS
- JavaScript
- web
---
My coworker Kristján and I recently ran into an interesting situation; one of
those problems where the solution made you ask yourself why you had the problem
in the first place.

In a nutshell, we were attempting to send an AJAX <b>POST</b> request from
a page loaded over HTTP (`http://mywebsite.com`) back to
our server over HTTPS
(`https://mywebsite.com/postback_path`) using JavaScript. While
testing, we were surprised to find out that our <b>POST</b> requests were
being transformed into <b>OPTIONS</b> requests (if you're not familiar, an
HTTP <b>OPTIONS</b> request asks the server to simply return a list of all
methods the particular resource supports in the response headers). These
<b>OPTIONS</b> requests were obviously not what we were expecting. The
important question, of course, was why were our requests being transformed?

# The Problem

It turned out that we were running into a classic problem of the [same origin
policy](http://en.wikipedia.org/wiki/Same_origin_policy), a security policy
dictating which resources browser scripts are able to access.  The problem was
that we were attempting to <b>POST</b> to a resource using a _different
protocol_ from that of the page from which the script was loaded.

At first, this seemed incredibly stupid to us. We could see the reason why you
would want to prevent the inverse from occurring (i.e. sending a <b>POST</b>
request over HTTP from a page originally loaded over HTTPS), but not for our
situation.  The former was bad because you were sending unencrypted data over
the wire from a page which any user would expect to be secure, but the latter
involved no such "loss" in security.

Upon deeper reflection, the decision for the same origin policy to act this way
was probably more a by-product of the fact that the protocol could be FTP, or
SSH, or any number of other protocols which the user would not expect to be
invoked when visiting a page served over HTTP. Including an exception for HTTP
and HTTPS so that our use case would work would make the specification more
nuanced.

However, we were still stuck with the problem of sending some data back
to our server securely, and so we needed a workaround.

# Proper Solutions that Wouldn't Work (For Us)

While the obvious answer would be to convert the whole site to use HTTPS so
that there is no change in protocol (and there are plenty of reasons why that
should just be the default approach), this was a more involved change than we
had time for, as the particular feature we were trying to implement was
time-critical (aren't they always?).

The other solution was to add [cross-origin resource
sharing](http://www.w3.org/TR/cors/#access-control-allow-origin-response-hea)
support, which is done by adding <b>Access-Control-*</b> headers to the
response returned by the server.  These headers specify whether or not
resources returned from the server can make cross-origin requests to other
resources.

The problem with this solution? The client has to respect the headers, and
wouldn't you know it, [IE7 does not support them](http://caniuse.com/cors).
Given the number of users that (unfortunately) still use outdated software, we
had to find another way.

# A Hack That Worked

Given our increasing frustration with trying to find a solution, we became
desperate. What we were trying to accomplish seemed like such a simple thing to
do, and yet we were having so much trouble with so trivial a task. It's at this
point where greater men may have prevailed, but we decided to throw caution to
the wind and hack together something quick.

Our solution? Send the data back to the server by loading an "image" from the
server over HTTPS, with the query parameters of the data we wanted
to send in the path.  The server would interpret and record the query parameters
in the request and then serve a 1x1 transparent image in response. We dropped
the following HTML onto our page:

{% highlight html %}
<img src="https://mywebsite.com/postback_img?param=..." />
{% endhighlight %}

I hope this idea makes you sad, because it sure made us sad. It made us even
sadder when it worked. The fact that we were seemingly able to violate the
principles of the same origin policy outside of JavaScript was silly. What was
the purpose of the same origin policy if it could be broken? It seemed that the
problem that got us into our situation was solved by simply ignoring the
problem. That's not supposed to work in the real world!

In any case, the solution worked. The obvious disadvantage was that our
<b>POST</b> request had to become a <b>GET</b> (since images are only loaded
using <b>GET</b>). A small price to pay to finally put this problem to rest
(kinda). IE 7 can't die soon enough.
