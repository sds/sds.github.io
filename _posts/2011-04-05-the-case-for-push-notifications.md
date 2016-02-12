---
layout: post
title: The Case for Push Notifications
tags:
- email
- Mozilla
- notifications
---
If you haven't heard of push notifications, you've either been living under a
rock, are already using them and didn't know it, or are about to start using
them really soon. "Push" notifications, for those unfamiliar, is simply a term
used to describe client-server communication where the server initiates the
transfer of a message---in this case, a notification. This is in contrast to
pull technologies, in which the client must poll the server for updates. Most of
our digital life today is relies on being notified by the web services we use.
Email has been the primary medium used for this role, but it is not the ideal
solution.

# Why Does Email Suck for Notifications?

There are a number of reasons why we need an alternative to email, specifically
for notifications:

## Too many emails are notifications

The problem with email is that we get too many emails, and a lot of these emails
aren't terribly useful. Whether receiving a confirmation for a website
registration, changing a password, or being told that our bank statement is
ready, email has become the de facto method of receiving notifications. When
these notifications start to fill up your inbox, it requires more effort to
filter the useful from the useless, as well as making sure not to miss important
messages (such as that reminder about your mother's birthday tomorrow).

## Email is slow

You probably don't need to be reminded, but email is a slow communication
medium. Not in the sense that it takes a long time to send a message (although
indeed email was not designed for real-time communication), but in the sense
that working with it requires a significant time investment to keep your inbox
organized and useful instead of a pile of junk. At its core, email is an
_active_ method of communication, as it demands your conscious attention,
and thus your time, in order for it to be useful.

## Email is too open

A major problem with email is that it is _open_; given your address, anyone can
send you messages using that address. While this convenience is one of the main
benefits of using email, the realities are that the moment you disclose your
email to someone or some service, you are placing your trust in that entity not
to share it with someone else. The moment they break this trust, you begin
receiving unsolicited email. That's a scary thought, and it can make people
hesitant to disclose their personal email to web services with which they have
no currently existing relationship. Another problem is that most people's
emails directly link to their identity in some way (especially professional
emails such as john.doe@company.com) Given these two problems, some individuals
resort to owning two emails: a personal email, and a “junk” email for
registering with websites. There are also [services][33M] that provide
disposable emails which can have messages forwarded to your main email, but
this still requires that you trust that service with your email in the first
place. Thus while there are workarounds, they are not particularly ideal.

# What Does a Solution Need to Have?

While email isn't going away anytime soon, there is definitely a need for a
separate channel when it comes to notifications. Any solution that claims to
solve the problems with email notifications at a minimum has to address the
following issues:

## Notifications should not require user action

The problem with email is that if you get a notification for something you
aren't interested in, it still occupies your inbox until you delete it, or at
minimum mark it as read. As was mentioned before, email is an active
communication medium in that it requires your full attention to manage. A proper
solution should make this process more passive, such that the notification
should not require explicit user intervention.

## Subscribing to notifications should not require personal information

With many users owning email addresses that contain information associated with
their actual identity, there needs to be a way to subscribe without giving up
this identity. Obviously, there are exceptions where this limits the usefulness
of the service, but the information should be volunteered rather than required.
For example, when registering for something like Facebook notifications, you
have already volunteered in signing up for the service, and now are just
requesting to use an additional feature. For a random blog however, there should
be no need to require personal information in order for a user to receive
updates.

## Users should have full control over the privilege of sending them notifications

Empowering users with the ability to revoke notification subscriptions will give
them more confidence in signing up for website notifications, as they know that
they will always be able to turn off those notifications should they become a
nuisance. Furthermore, keeping this power in the hands of users will force
providers to make sure they are only sending out notifications that are useful
to their users, or face the risk of having their users unsubscribe.

# Isn't It Already Solved?

The short answer is no. While there are indeed products out there that exist as
push notifications solutions ([Apple's Push Notification Service][APNS] and
[Android's Cloud to Device Messaging Framework][ACDMF] are two of the larger
platforms that come to mind) they all suffer from a fundamental problem of
being centralized.  These two issues materialize as serious problems because
they tie users to one platform. This is unhealthy for an open platform such as
the web, since it forces users to choose between factions. It also makes it
difficult for developers, as they must now develop push notifications support
in their apps for each and every platform.

The success and ubiquity of email is largely due to the fact that it is a
decentralized protocol---anyone can run their own mail server, and anyone can
send mail to any other server. In order for notifications to really grow and
become useful to users, they need to be able to receive those notifications
anywhere, on any platform; it needs to be a decentralized model.

The big question is: can it be done? I think it can, and the answer is with
Mozilla Push Notifications, which I'll talk about more in a future post.

[33M]: http://33mail.com/
[APNS]: http://developer.apple.com/library/ios/#documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/ApplePushService/ApplePushService.html
[ACDMF]: http://code.google.com/android/c2dm/index.html
