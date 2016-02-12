---
layout: post
title: Git Shortlog and Mailmap
tags:
- Tools
- git
---
I stumbled across an interesting feature of `git` the other day. It's called
the shortlog, and if you haven't heard of it, now is as good a time as ever.

In a nutshell, the shortlog allows you to view a summary of all the commits
made by all users to a repo. For large repositories, this doesn't sound
immediately useful, as this will just print out thousands upon thousands of
commits, organized by user. However, the real power of the shortlog lies in its
ability to summarize this information. Executing `git shortlog -nse` will
display the number of commits for each committer along with their name and
email in descending order:

<output title="Condensed output of `git shortlog -nse`">
   102  Steve McQueen &lt;steve@mcqueen.com&gt;
    87  Paul Newman &lt;paul@newman.com&gt;
    57  Humphrey Bogart &lt;humphrey@bogart.com&gt;
</output>

What I like about this is that it gives contributors an opportunity to see what
kind of impact they have made on the repository as a whole. It can also instill
a bit of friendly competition into committing code, and better yet, encourage
authors to make smaller, more atomic commits in order to get their count up.

There is however, an unfortunate case that often occurs. If one of the
repository's contributors commits changes using multiple author names or emails
(when on a different machine for example), their entries in the shortlog will be
spread over multiple authors.

<output title="Output of shortlog before using .mailmap">
    87  Paul Newman &lt;paul@newman.com&gt;
    60  Steve McQueen &lt;steve@mcqueen.com&gt;
    57  Humphrey Bogart &lt;humphrey@bogart.com&gt;
    42  S. McQueen &lt;steve@oldqueen.com&gt;
</output>

This is obviously a problem, as it doesn't give the author the credit they
deserve, since they appear to have contributed fewer commits than others upon
first glance. What would be nice is if those two author/email pairs could
coalesce together and be seen as belonging to the same author.

# The .mailmap File

Well, `git` being the wonder it is allows you to do this by adding a file
called `.mailmap` in the top-level directory of your repository.  Similar to
`.gitignore`, `.mailmap` is just another configuration file recognized by
`git`. The difference is that `.mailmap` allows you to non-permanently
"rewrite" the commit author history so that old emails or author names map to a
single canonical author name. The `.mailmap` file itself is simply a series of
lines in the following format:

<output title="Line format of .mailmap file">
Canonical Author Name &lt;canonical-author@email.com&gt; Commit Author Name &lt;commit-author@email.com&gt;
</output>

Using our example above, say we wanted to have `S. McQueen` appear as `Steve
McQueen`. This is done by adding the following line to our `.mailmap` file:

<output title="Example line in .mailmap file">
Steve McQueen &lt;steve@mcqueen.com&gt; S. McQueen &lt;steve@oldqueen.com&gt;
</output>

At which point if we run `git shortlog -nse` we'll get the following:

<output>
   102  Steve McQueen &lt;steve@mcqueen.com&gt;
    87  Paul Newman &lt;paul@newman.com&gt;
    57  Humphrey Bogart &lt;humphrey@bogart.com&gt;
</output>

Note that while the `.mailmap` file format does allow you to take shortcuts by
omitting certain fields such as the commit author name or email, this will make
your mapping too general and can potentially bite you in the future. Why?
Suppose Bob Dole uses the email `bob@company.com`, makes a few commits, and
then leaves the company. If a new employee Bob Hope comes along and starts
commiting with Bob Dole's old email, using a simplified mail mapping will cause
Bob Dole's old commits to be incorrectly attributed to Bob Hope. Thus it is
best to be explicit and specify all the fields when mapping authors and emails.

# Creating a .mailmap File for an Old Repository

If you're starting out with an old repository, setting up a `.mailmap` file can
at first appear daunting. Having to cross-reference all the author name/email
pairs and figuring out what the canonical email is can be painful and time
consuming. This bothered me, so I came up with a script to get the hard work
done for you. Run the following in the top-level directory of your repository:

{% highlight bash %}
git shortlog -se | \
  awk -F'\t' '{print $2,$3,$2,$3}' | \
  sort > .mailmap
{% endhighlight %}

This script first gets the commit count summary for all author name/email pairs
as before. It then pipes it into `awk` (the `bash` handyman's ducktape) which
pulls out the author name and email, printing each twice so that each line
resembles a line of the `.mailmap` file. Finally, it sorts the output so that
similar names are placed together (making it easier for a human to notice
similarities) and writes it to the `.mailmap` file. Running this script on our
unmapped example above, we'll get:

<output title="Output of helper script">
Humphrey Bogart &lt;humphrey@bogart.com&gt; Humphrey Bogart &lt;humphrey@bogart.com&gt;
Paul Newman &lt;paul@newman.com&gt; Paul Newman &lt;paul@newman.com&gt;
S. McQueen &lt;steve@oldqueen.com&gt; S. McQueen &lt;steve@oldqueen.com&gt;
Steve McQueen &lt;steve@mcqueen.com&gt; Steve McQueen &lt;steve@mcqueen.com&gt;
</output>

With this output we can easily see the similarity of `S. McQueen` to `Steve
McQueen`, and change the name and email accordingly so it gets mapped to the
canonical `Steve McQueen`. Not too shabby!

At the end of the day, these changes will only affect the output of `git
shortlog` (you won't see the updated author info when you go digging through
`git log`, for example). However, that doesn't change the usefulness of having
a clean shortlog. I've personally found `git shortlog -nse --no-merges` to be
useful enough that I've aliased it to `git committers` in my `.gitconfig`.
