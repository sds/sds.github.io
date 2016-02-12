---
layout: post
title: Git Branch Tracking
tags:
- git
- tools
---
I use [`git`][GIT] on an everyday basis---it's a powerful tool that supports
many different workflows. However, depending on your workflow, there are useful
configurations you can set up that make using the tool that much easier.

`git` really shines when you can make the UI give you a little more information
about what's going on as you work. This can be done with a built-in feature of
`git` called tracking branches.

# Tracking Branches

All (well, almost all) branches in git "track" a branch. Usually this branch is
a remote branch, like `origin/master`. However, you can also set up tracking of
local branches.  One way you can specify that your current branch should track
another branch is to run the following:

{% highlight bash %}
git branch --set-upstream `git rev-parse --abbrev-ref HEAD`
{% endhighlight %}

This gives you the ability to compare the current branch relative to the branch it
is tracking, for example, if you are on branch `master` and it is tracking
`origin/master`, when you make a commit and run `git status` part of the output
will say:

<output title="Output of `git status` with a tracking branch specified">
Your branch is ahead of 'origin/master' by 1 commit.
</output>

`git` determines this by finding the common ancestor between the current branch
head of `master` and the head of tracking branch `origin/master`. Since `master`
has only one commit between its head and the head of `origin/master`, it returns
the above output.

Although rarely used, I've found it useful to create a `git track` alias by adding
the above command to the list of aliases in `~/.gitconfig`:

{% highlight ini %}
[alias]
  track = !sh -c 'git branch --set-upstream `git rev-parse --abbrev-ref HEAD` $1' -
{% endhighlight %}

This is a little more user-friendly and intuitive for those cases where I explicitly
want to set a tracking branch.

Another useful shortcut is a `git tracking` alias which displays which branch a
specified branch is tracking. This is slightly more complicated to do and thus isn't
a one-liner, but here's the shell script:

{% highlight bash %}
#!/usr/bin/env bash

if [ -n "$1" ]; then
  local_branch="$1"
else
  local_branch=`git symbolic-ref -q HEAD | sed 's/^refs\/heads\///'`
fi

remote_branch=`git config branch.${local_branch}.merge | sed 's/^refs\/heads\///'`
remote=`git config branch.${local_branch}.remote`

if [ "$remote" == "." ]; then
  echo ${remote_branch}
else
  echo ${remote}/${remote_branch}
fi
{% endhighlight %}

If you save this as `git-tracking` and give it execute permissions, you'll be able to
run it using `git tracking`.

Notice that if a branch is tracking a local branch (i.e. a branch local to this repository)
instead of a remote branch (i.e. a branch on `origin`), the name of the remote
is just `.`, or in other words the current repository (if you think of `.` as the
similar to `ls`'s representation of the current directory, it makes sense).

This script works by extracting the tracking branch information using `git config`, which
itself simply reads from the local `config` file in the `.git` directory for the repository.
Each branch in your repository that has been set to track another branch will have an entry
in `.git/config` that looks something like the following:

{% highlight ini %}
[branch "master"]
  remote = origin
  merge = refs/heads/master
{% endhighlight %}

The key points to observe are that this configuration states the local `master` branch
tracks a remote repository called `origin`, and the branch on it tracks on
`origin` is called `master` (thus, `origin/master`).

# Set Tracking Branches Automatically

A slightly odd problem with tracking branches is that they aren't set up
automatically when creating a new branch (except for `master` being set to
track `origin/master` when you first clone a repository). However, this
behaviour can be changed so that when you create a new branch it tracks the branch
you're currently on, or the remote branch you pulled from. Add the following
to your `~/.gitconfig`:

{% highlight ini %}
[branch]
  autosetupmerge = always
{% endhighlight %}

The `branch.autosetupmerge` setting dictates whether or not `git` sets up new
branches so that they will be mergeable with the starting-point branch (usually
the branch you were on before creating the new branch). It basically saves
you from having to run the `git track` alias discussed above.

# How Does This Help?

While tracking branches are relatively intuitive to understand, they don't seem
terribly useful as a concept so far. After all, `git` doesn't automatically set
up these tracking branches in all situations, so why bother making the above
change to _always_ set up tracking branches, even for local branches?

This will become apparent in a follow-up post, where I'll discuss the power of a
[rebase-based `git` workflow][RBB].

[GIT]: http://git-scm.com/ "Git: distributed source control management"
[RBB]: /2013/01/28/rebase-based-workflow.html
