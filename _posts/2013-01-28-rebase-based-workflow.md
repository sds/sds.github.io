---
layout: post
title: Rebase-based Git Workflow
tags:
- git
- tools
- workflows
---
[`git`][GIT] has seen such a huge adoption over the past few years, so it's
no wonder that there have emerged a number of suggested workflows for it.
What `git` lacks in user-friendliness, it certainly makes up for in
flexibility, and it's this flexibility that has resulted in this wide range
of possible use cases.

I've spent a while working with `git` myself, and during that time I've seen
a few workflows. It's obvious that different workflows apply for different
situations---the [recommendations for working within the `git` source][GWF]
itself are based on the fact that multiple versions need to be maintained,
and thus there is a "graduation" system where changes are first merged into
the oldest branch requiring support, and then gradually merged upwards to
later branches, until a commit finally makes it onto the master branch.

This flow doesn't really apply to most, and it's not entirely unreasonable
to say that a lot of the features of `git` are unnecessary for the typical
developer. That's the wonder of `git`, though: you can customize it to suit
your needs.

# Workflow

At [Causes][CAU], we use `git` in a perhaps unconventional---but simple---way.
Whenever starting on a task, a new branch is created to encapsulate all the
changes as a part of that task (you might have heard of this referred to as a
"topic" or "feature" branch). As you work on it, the canonical source
(`origin/master`) will be updated by other developers, and so we rebase that
feature branch on top of `origin/master` often so any conflicts can be
addressed earlier rather than later (this is in contrast to the merge option,
where you use a merge commit to address any conflicts, dealing with them only
at that point in time). After commits are approved via our code review tool
[Gerrit][GER], the commits are cherry-picked serially onto `origin/master`.

<figure>
  <img src="/images/posts/git-flow-at-causes.png"
       title="Git flow at Causes"
       width="80%">
</figure>

Conceptually, this means our commit history is perfectly linear, as any chunk
of work is simply a series of commits that is appended to the history when it
is merged onto master. There are no "bubbles" or branches of any kind in the
output of `git log --graph`.

<figure>
  <img src="/images/posts/linear-vs-merge-based-histories.png"
       title="Linear versus merge-based histories"
       width="50%">
</figure>

Experts might find this to be a great injustice, as you lose information about
where branches started and when they were merged back in, but we've found this
to not be a significant loss of information. We believe the true value of a
repository history lies in well-written commit messages. We also believe that
long-lived topic branches are to be avoided, as we want to aim to always push
something to master at the end of every day.

# Topic Branch Dependency

A situation that can sometimes arise is when you are working on a topic
branch that builds on top of another topic branch which has not yet been merged
onto `origin/master` (due to pending code review, for example).

<figure>
  <img src="/images/posts/topic-branch-dependency.png"
       title="Topic branch dependency"
       width="35%">
</figure>

In order to continue being productive, you want to continue working on
`feature-2`, but it's likely that you'll have to make some changes to
`feature-1` after feedback from code review, or you'll have to rebase
`feature-1` onto a new `master` after fetching `origin/master`, or both.

This requires you to rebase each branch serially in order to preserve the
references of each branch. To see why, consider what would happen if you were
to just rebase `feature-2` onto an updated `master` using `git rebase feature-2
--onto master` (notice in the figure below that the colours of the commits are
modified to emphasize that their hashes have changed from the rebase):

<figure>
  <img src="/images/posts/straight-up-rebase.png"
       title="Before and after straight-straightforward rebase"
       width="70%">
</figure>

Notice that the commit `feature-1` is pointing to is no longer the predecessor
of `feature-2`, which during the rebase had its parents updated to the same
commits as `feature-1` but with different hashes due to the rebase. To fix
this, you would need to manually set `feature-1` to point to the new commit
(`git checkout feature-1 && git reset --hard feature-2~2`).

Alternatively, you could avoid this problem by rebasing each branch in serial,
starting from `master`:

<code class="block" lang="bash">
git fetch origin
git rebase master --onto origin/master
git rebase feature-1 --onto master
git rebase feature-2 --onto feature-1
</code>

# Managing Large Dependency Chains

This becomes annoying in the case where you have very large dependency chains,
as you have to traverse the chain up to master and then recursively rebase each
branch onto its rebased parent in serial.

<figure>
  <img src="/images/posts/long-branch-dependency-chains.png"
       title="Long topic branch dependency chains"
       width="20%">
</figure>

Although this is a relatively rare occurrence, I was intrigued by the idea of
automating this task. I call it a _recursive rebase_, and I use it regularly as
part of my workflow. It's powered by the following recursive script:

{% highlight bash %}
git-recursive-rebase () {
  if ! git diff-index --quiet HEAD; then
    echo "You have local changes; either stash them or get rid of them"
    exit
  fi

  local old_head=`git rev-parse HEAD`
  local local_branch=`git symbolic-ref -q HEAD | sed 's/^refs\/heads\///'`

  if [ -z "${local_branch}" ]; then
    echo "HEAD is currently detached--aborting."
  fi

  local remote_branch=`git config branch.${local_branch}.merge | sed 's/^refs\/heads\///'`
  local remote=`git config branch.${local_branch}.remote`

  # If remote is local repository, then we're tracking a local branch
  # We want to fetch that branch's remote branch, so change to it and recurse
  if [ "$remote" == "." ]; then
    # Switch to tracking branch, refresh it, and switch back to our branch
    git checkout "${remote_branch}"
    git-recursive-rebase
    git checkout "${local_branch}"
  fi

  if [ "${remote}" == "." ]; then
    local rebase_ref="${remote_branch}"
  else
    git fetch ${remote}
    local rebase_ref="${remote}/${remote_branch}"
  fi

  if [ -n "`git rev-list ${local_branch}..${rebase_ref}`" ]; then
    echo "Rebasing ${local_branch} with changes from ${rebase_ref}"
    if ! git rebase ${rebase_ref}; then
      echo "##################################################################"
      echo "UNABLE TO REBASE"
      echo "Run 'git rebase ${rebase_ref}' to deal with this conflict manually"
      git rebase --abort
    fi
  fi
}
{% endhighlight bash %}

In a nutshell, the script starts on the current branch, and recursively
traverses the current branch's tracking (i.e. remote) branch until it finds a
tracking branch whose remote is actually a remote repository (e.g. `origin`
instead of `.`, which is used to represent the local repository).

I've set the alias `git rrb` to execute this script, completely automating this
process for me. There's probably some edge cases I've missed, but after having
used it for a while it's worked without any surprises. It may be
over-engineered, but it's pretty darn cool.

[GIT]: http://git-scm.com/ "Git: distributed source control management"
[GWF]: http://www.kernel.org/pub/software/scm/git/docs/gitworkflows.html
[CAU]: https://www.causes.com/
[GER]: http://code.google.com/p/gerrit/ "Gerrit Code Review"
