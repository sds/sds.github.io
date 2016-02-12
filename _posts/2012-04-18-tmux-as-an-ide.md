---
layout: post
title: tmux as an IDE
tags:
- Tools
- tmux
- vim
---
I started using [`tmux`](http://en.wikipedia.org/wiki/Tmux) almost year ago. In
that time, I've seen amazing improvements in my development efficiency from the
command-line---so much so that I don't think I could go back to a regular IDE
anytime soon. The `tmux` + `vim` combo is unbeatable, and has become my new
IDE.

There's been a lot of buzz amongst the hacker community about the wonders of
`tmux` as a productivity tool. The following are my own thoughts on what makes
it such a great asset to a developer.

# Consistency

Using `tmux` with `vim` brings a level of consistency that I honestly didn't
think was possible with command-line tools. One of the reasons GUI-based IDEs
are so popular is that they present a single cohesive view of a project space.
From one application you can view the source of a project, build the project,
debug it, and run it. It makes for less context-switching amongst your tools,
allowing you as a developer to achieve that "flow" where you are most
productive.

Despite being a command-line tool, `tmux` allows you to achieve this level of
flow. Out of the box, its ability to do this is _ok_ but with the right amount
of customization you can truly make it shine. Adding simple shortcuts for
jumping between window panes allows you to navigate between source code and
command-line in a split second, making the time between making a change and
testing it incredibly short. Furthermore, adding shortcuts for easily splitting
window panes or adding new window panes (or tabs) makes it easy to split up a
project into various portions as you work on it.

I've customized my `tmux` shortcuts to be analogous to those of `vim`. This way
navigation, copy/pasting, and other common actions are consistent throughout my
use of the tool. It makes it feel like I'm using a single piece of software,
rather than a bunch of open source tools thrown together. This consistency is a
pleasure to work with, and is the main reason I don't miss using regular IDEs.

# Workspace Management

It's rare today to find yourself working on a single project. Having a tool
that allows you to manage multiple projects without itself becoming a burden
can be difficult. I've found `tmux` lends itself well to workspace management
through the use of sessions. By keeping a separate session for each project I'm
working on, I can easily switch to another project by connecting to another
session.

Within the terminal this usually results in me opening a new tab for each
project and starting a session for that project. In a session, I may split a
pane into a view for source code and a small shell at the bottom, or I may
create a separate pane to display the output of logs or other useful
information to check from time to time.

The other great side-effect of this approach is that all of my workspace
"contexts" are saved, so that when I go back to them everything is just how I
left it. Especially when I'm working on an always-on remote machine, I can
relax knowing that should I lose my connection or that I've just had enough and
leave, my project will be waiting for me when I come back.

# Flexibility

When it comes to learning a new language or development stack, having an IDE
handle the nitty-gritty build details and project management takes a huge
weight off a newbie's shoulders. However, there comes a point when said newbie
outgrows his training wheels, and finds that the nurturing environment their
IDE offered was overly constraining. There is usually only "one way" to
organize a project with an IDE, and it's difficult to work with other
developers unless they are using the very same IDE.

Most open source projects aren't organized in a manner immediately loadable by
IDEs like Visual Studio or NetBeans. Usually you download a repository of some
kind, run `make`, and away you go.  It's done this way so that people can
choose which development environment they want to use, rather than be forced
into one. There's also less bloat in terms of auxillary files that need to be
checked in or tracked by source control. While `tmux` is awesome, using it when
developing your projects doesn't force you to use `tmux` for the rest of the
project's existence.

In the end, development with `tmux` has made my life easier, and I would
encourage you to think about incorporating it into your own workflow.
