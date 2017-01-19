---
layout: post
title: Sobbing with Ruby File Globing
tags:
- unix
- ruby
---

[Globbing][GLOB] is a term used to refer to a pattern matching language,
usually used in UNIX environments to search for files with a concise syntax.
They do not possess the same flexibility as [regular expressions][REGEX], but
are often good enough to get the job done.

They're also really confusing if you try to use them in Ruby.

# A Quick Recap

As a reminder, here are a basic set of glob patterns for a typical UNIX shell:

* `*`: match any number of any characters
* `?`: match any single character
* `[AB]`: match any of the characters in the set (`A` or `B`, in this case)

Unfortunately, as was already alluded to above, different languages and
platforms have slightly different implementations/definitions for globs. This
can start to make your life frustrating when trying to use them.

# Oh My Stars

I recently ran into an issue where Ruby's implementation of glob patterns
wasn't giving me the results I expected when using a double asterisk
(`**`). From the [Dir.glob documentation][DIRGLOB], `**` is supposed to
match directories recursively, leading one to assume you can use it to
represent "zero or more directories".

Consider the following directory structure:

{% highlight bash %}
cat.js
dog.rb
directory/
  mouse.js
  fox.rb
{% endhighlight %}

If you were to run `Dir.glob('**/*.rb')` from an `irb` console in this
directory, you'd get the following output:

{% highlight bash %}
=> ["directory/fox.rb", "dog.rb"]
{% endhighlight bash %}

In this case, the behaviour of `**` matches our expectation given the
documentation, in that it finds all files ending in `.rb`, regardless
of how many directories they are nested within.

Ruby also has `File.fnmatch?`, which allows you to perform glob matching
against strings. This saves you from having to scan the file system like in the
case of `Dir.glob`.

What happens if we find all files that match the glob using `File.fnmatch?`?
Note we're using `Dir.glob` just to get all the files in the directory so we
can compare that list against what would be matched if we used `File.fnmatch?`.

{% highlight ruby %}
all_files = Dir.glob('**/*')

all_files.select { |f| File.fnmatch?('**/*.rb', f) }
{% endhighlight %}

{% highlight bash %}
=> ["directory/fox.rb"]
{% endhighlight %}

While we would expect the output to be identical, `File.fnmatch?` didn't
include files from the root directory! It seemed to only find files that
were nested within at least one directory. Why?

It turns out that there are flags that `Dir.glob` and `File.fnmatch?` accept
which change the behaviour of the glob matching, and for some reason they don't
have the same defaults (this is true in Ruby 2.1.2, the latest version at time
of writing). Flags of note are:

* `FNM_DOTMATCH`: Allow `*` to match filenames starting with `.` (i.e. dotfiles)
* `FNM_EXTGLOB`: Enable globbing via `{a,b,c}` (match `a`, `b`, or `c`, etc.)
* `FNM_PATHNAME`: Prevent wildcards from matching path separators

For the example above, we can get `File.fnmatch?` to produce the same output by
enabling the `FNM_PATHNAME` flag. If you want your globs in Ruby to act like globs
in a typical UNIX shell, you'll want to enable the `FNM_EXTGLOB` flag as well.

# Ruby's Globbing in a Nutshell

The `**` shenanigans described above is really just a Ruby nuance, as most glob
implementations get the job done with `*`. Ruby complicated the issue by
providing a way to change the behavior of wildcards such that matching a path
separator could be toggled.

Furthermore, not having the default behavior for both `Dir.glob` and
`File.fnmatch?` match each other is a significant source of confusion.
Unfortunately, this area of Ruby's documentation and code isn't very well
documented, so it's difficult to tell what the original intention was.

At the end of the day, you can work around the problem by always explicitly
passing in the same flags to both `Dir.glob` and `File.fnmatch?`.

[GLOB]: https://en.wikipedia.org/wiki/Glob_(programming)
[REGEX]: https://en.wikipedia.org/wiki/Regular_expression
[DIRGLOB]: http://ruby-doc.org/core-2.1.2/Dir.html#method-c-glob
