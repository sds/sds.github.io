---
layout: post
title: Breaking Internet Law
tags:
- DNS
- dynamic DNS
- ZoneEdit
---
I purposefully broke Internet law today...well, sort of---but I did it with
good intentions, I swear!

It all started so simply. I was trying to get a web server running at home so I
could host some stuff for a project I'm currently working on (more on that in
another post). I already owned a domain name I wanted to use for this server---
we'll call it `flubber.com`---and simply wanted to have pages from my web
server appear when I typed `flubber.com` into my browser.

Now, anyone who owns a domain name knows that in order to point your domain to
some server, that server needs to have an identifier that doesn't change---in
other words, an IP address. The problem is that my router's public IP address
changes every once in a while, just like 95% of Internet users with broadband.
I had no desire to go purchasing a static IP from my ISP, so I told myself
there must be a better way.

# Dynamic DNS

Well there was, and the answer was with dynamic DNS. Using a service such as
`dyndns.org`, I was able to sign up for a free account and get a free
subdomain---we'll call it `flubber.dyndns.org`---whose IP address would map to
my router's IP address. Any time my ISP assigned my router a new IP, the router
would then be able to send the new IP to `dyndns.org`, ensuring that the
address was always kept up to date. This meant I could access my server from
anywhere using `flubber.dyndns.org`, absolutely free.


However, using `flubber.dyndns.org` as my domain wasn't ideal---I wanted to use
`flubber.com` as my domain. I figured I would just be able to add an alias in
my DNS configuration that would point `flubber.com` to `flubber.dyndns.org` and
that would be the end of it.  To my chagrin however, my domain provider at the
time---we'll call them SlowDaddy---wouldn't let me do this, and the error
message I received wasn't terribly useful in explaining why.

# You "Can't" CNAME a Naked Domain

Looking into this, I soon discovered that what I was attempting to do was
breaking [RFC 1034][RFC1304], section 3.6.2. In a nutshell, you are not allowed
to have a `CNAME` resource record for a what's called a _naked domain_ (for
example, `flubber.com` is a naked domain, but `www.flubber.com` is not). The
reasoning is a bit nuanced, but it is indeed valid for some cases.

This was disappointing, as while I sort of understood why this could potentially
be bad (the name server might have a problem if the data specified by other
records and the canonical name don't match), it didn't really apply in my case.

# Not Everyone Cares If You Break the Rules

After a bit of hunting, I discovered a website [ZoneEdit][ZE] that allowed me
to get around this restriction. Not only was its interface far better than
SlowDaddy's DNS manager, but when I tried creating a `CNAME`
`flubber.dyndns.org` for `flubber.com`, it didn't complain. Since SlowDaddy
still manages `flubber.com`, I had to change the name servers from SlowDaddy's
to ZoneEdit's, but once that was done, within a matter of minutes I was able to
enter `flubber.com` in my browser and see my web server respond. Mission
accomplished!

At the end of the day, the only "proper" solution to this problem is to create
an `A` record in your zone file that points your domain name to a static IP
address. If however, you're just fooling around, this solution is perfectly
acceptable, although you'll have to live with the fact that you are directly
violating an RFC published by the [Internet Engineering Task Force][IETF]. I'm
all for open standards, and I think it's important that organizations like the
IETF exist. However, so long as I'm not hurting anyone in my own little
projects, I think this is one violation that can be forgiven.

[RFC1304]: http://www.ietf.org/rfc/rfc1034.txt "RFC 1034: Domain Names - Concepts and Facilities"
[ZE]: http://zoneedit.com "ZoneEdit DNS Zone Management"
[IETF]: http://www.ietf.org/ "Internet Engineering Task Force"
