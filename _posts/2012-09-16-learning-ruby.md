---
layout: post
title: Learning Ruby
tags:
- Ruby
- Haskell
- Shell
- scripting
- functional programming
- metaprogramming
---
Like natural languages, learning a new programming language and mastering the
frameworks built on top of it allows you to think differently about how to
think---in the case of developers, changing how you think about problems you've
already solved, and affecting how you think about problems in the future. For
me, these alternative perspectives sometimes give new insights into the problem
itself, and tend to change the way I think about solving problems of that
nature. I'd like to think it's for the better.

The language I've been privileged to learn for the past year is [Ruby][RB], so
aptly named as it is indeed a gem of a language. Given my relatively brief
exposure, if I had to describe the language it would be an "object-oriented
functional-influenced reflective programming language", though I'm sure the
masters would have plenty more to add. Coming from a [Python][PY] background,
I've found the language pretty easy to pick up. This post attempts to summarize
some of my thoughts on what makes Ruby great.

# Everything is an Object

Perhaps the first concept any newcomer will learn is that in Ruby, everything
is an object. There are no primitives. Everything (yes, even integers or
booleans), descends from the `Object` class. This may seem odd, as one would
imagine that allocating a whole object for each primitive is wasteful (this
doesn't actually happen, thankfully, but it's still initially an odd feeling).
However, once you consider that Ruby was created for programmer productivity,
this makes a lot of sense. Ruby's "everything is an object" philosophy seems to
be the stepping stone from which the rest of the language's design was based,
and makes the language very consistent.

# Everything Returns Something

In Ruby, every statement returns something. Whereas other languages usually
require some sort of explicit `return` statement or temporary variable to hold
a value so it can be read later, Ruby allows you to be really concise
without losing readability. For example, consider a `switch` statement in
Java (this is a silly example for illustrative purposes only):

{% highlight ruby %}
public String monthToInt(month) {
  int num;
  switch (month) {
    case "Jan":
      num = 1;
      break;
    case "Feb":
      num = 2;
      break;
    ...
  }
  return String.format("%s is month #%s", month, num);
}
{% endhighlight %}

In Ruby, you could write this as:

{% highlight ruby %}
def month_to_int(month)
  num = case month
  when 'Jan'
    1
  when 'Feb'
    2
  ...
  end
  "#{month} is month##{num}"
end
{% endhighlight %}

Notice how we were able to treat the entire `case` block as a single expression
and assign its value to a variable. Ruby's approach in this and other
situations like it makes for much more concise code without loss of
readability.

# Ruby is Self-Aware

Alright, we're not talking Skynet here, but Ruby boasts some features that
pretty much let a program reason about and construct itself during runtime.
This kind of [metaprogramming][MP] is really cool. For example, there comes a
time in many programs where a series of boilerplate declarations are required,
whether they be constants or even similar methods following a general template.
With traditional languages, one must declare each of these explicitly.

With Ruby, one can define what is called a [domain-specific language][DSL]
(DSL), more easily than what would be possible with most other mainstream
languages. For example, the testing framework [RSpec][RS] allows you to write
code like the following:

{% highlight ruby %}
describe User do
  describe '#name' do
    context 'when the user has no name' do
      let(:user) { User.new }

      it 'includes "User" somewhere in the anonymized name' do
        user.name.should include 'User'
      end
    end

    context 'when the user has a name' do
      let(:user) { User.new name: 'Steve' }

      it 'returns the user name' do
        user.name.should == 'Steve'
      end
    end
  end
end
{% endhighlight %}

This is perhaps more a case for RSpec and [Behaviour-Driven Development][BDD],
but what's important to notice is how this doesn't really read like typical
boilerplate test code. Instead, what we see is actual written language like
"describe", and "include", which when read actually sounds more like a natural
language than a programming language.

Under the hood, this is all powered by Ruby's powerful ability to metaprogram
itself, changing the syntax of the language to make it more useful.

# Imperative and Functional

[Functional programming][FP] has seen a burst of interest over the past few
years. Unfortunately, it's not the most intuitive to read, as humans naturally
think imperatively, and this seems to have hindered its adoption into more
mainstream use. I personally feel that while the functional paradigm is indeed
a powerful one, purely functional programs don't scale very well, and are much
harder to understand. I imagine that opinion will be met with significant
resistance, and I'm hopeful that I'll be proved completely wrong one day.

For now though, Ruby provides an excellent blend of functional and imperative
language features. One major difference I like to tote between Ruby and
[Haskell][HS] (a functional programming language) is that it reads much more
naturally while still allowing composition of functions. Consider the following
problem: how do you reverse a string of words?

With Ruby:

{% highlight ruby %}
sentence.split.reverse.join(' ')
{% endhighlight %}

With Haskell:

{% highlight haskell %}
unwords $ reverse $ words sentence
-- Or...
(unwords . reverse . words) sentence
{% endhighlight %}

With the Haskell version, you had to read the code from right to left, and
furthermore had to almost create a parse tree in your head in order to fully
understand what was going on. With Ruby, you simple read the code from left
to right as is natural, and applied each transformation until you obtained
the desired result. Ruby has an almost [pipeline-like][PL] interface for
transforming data.

Granted, this kind of one-off comparison isn't entirely fair, but it summarizes
what I think is a general advantage of Ruby code over Haskell for everyday use.

# Great for Scripting

If you've ever done any shell scripting, you've surely had frustrations with
the likes of `bash` or any other shell language. Ruby provides a higher-level
programming interface for shell scripts, but keeps around a lot of the idioms
familiar to shell scripters; it's shell scripting on steroids.

For example, if you've already got some shell script that does the job, but
you wanted to do some additional processing on it, you can easily just
integrate that code into a Ruby script by dropping it in some [backticks][BT]:

{% highlight ruby %}
#!/usr/bin/env ruby

ignoredWords = `cat stopWords.txt`.split("\n")
words = Set.new
`grep '+' -R .`.split("\n").each do |line|
  words.merge(line.split(' '))
end

words -= ignoredWords
puts words
{% endhighlight %}

While the above example isn't particularly interesting (and somewhat abuses
backticks for sake of example), it demonstrates how easy it is to throw a quick
and dirty script together for on-the-fly scripting. Furthermore, the dollar
sign variables like `$1`, `$?`, etc. work the same way in Ruby, making the
transition to using Ruby over a traditional shell language that much easier.

# There's Something for Everyone

In writing this post, I know I'm a little late to the show. Ruby has been
around for years, and you need look no further than the [vast number of
packages][RG] that have been written, or the success of the [Rails][RLS] web
framework to see that there's a lot Ruby has to offer. I would encourage you to
check it out.

[RB]: http://www.ruby-lang.org/en/ "The Ruby Programming Language"
[PY]: http://www.python.org/ "The Python Programming Language"
[MP]: http://en.wikipedia.org/wiki/Metaprogramming
[DSL]: http://en.wikipedia.org/wiki/Domain-specific_language
[RS]: http://rspec.info/ "RSpec Documentation"
[BDD]: http://en.wikipedia.org/wiki/Behavior-driven_development
[FP]: http://en.wikipedia.org/wiki/Functional_programming
[HS]: http://www.haskell.org/
[PL]: http://en.wikipedia.org/wiki/Pipeline_%28Unix%29
[BT]: http://en.wikipedia.org/wiki/Grave_accent#Use_in_programming
[RG]: http://rubygems.org/ "Ruby Gems"
[RLS]: http://rubyonrails.org/ "Rails Web Framework"
