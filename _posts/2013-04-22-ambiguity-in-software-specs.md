---
layout: post
title: Ambiguity in Software Specs
tags:
- grammar
- language
- specifications
---
[Natural language][NL] is a powerful tool for communication. When communicating
with each other we use the context of the current situation or topic of
discussion---the current weather, the time of day, the previous conversation
topics, etc.---to allow us to take shortcuts in our speech in order to convey
thoughts.

For example, in the previous paragraph you knew when I said "we" that I was
referring to human beings. This is because given the domain knowledge and
context you have about this writing, you (reasonably) assumed that it was
written by a human being. It is however possible to interpret "we" as referring
to a different group, such as all software engineers, or all mammals.

In situations where despite the context of the discussion the meaning of the
words are still open to multiple interpretations, we say that a word or set of
words is [ambiguous][AB]. Ambiguity is an inherent part of natural languages,
but is usually kept in check when the right amount of context is provided, as
that of most natural conversation. When there is more than one interpretation,
we tend to pick the one that is the most likely given the context.

This phenomenon presents a problem when we start defining software requirements
and specifications, as bad things can occur if interpreting an ambiguous
requirement or specification incorrectly results in building software that
doesn't do what was originally requested. It is crucial that software engineers
avoid this situation, as it can be extremely costly to fix if discovered too
late---those months (but hopefully just a few days in most cases) of
development building the wrong thing can't be completely recouped!

There are three possible approaches you have to address ambiguity in software
specifications:

* Learn to recognize ambiguity
* Learn to write less ambiguously
* Write in a formal, non-ambiguous language

The first two options will be discussed in depth. The last option is a very
interesting field of exploration, but in practice tends to require more work to
implement. Thus we'll be exploring the first two options by showing some
examples of ambiguous writing, describing the problem, and correcting the
problem. Much of the following comes from a course I took on requirements
specification and analysis at the University of Waterloo.

# Detecting Ambiguity

When ambiguity exists in writing, there are different possible underlying
causes of the ambiguity. These kinds of ambiguity are outlined below. Note
that in the academic literature there are a few more than are mentioned here,
but they start to blur the distinction between the different forms, and so they
have been omitted in the interest of keeping the conceptual differences clear.

## Lexical Ambiguity

[Lexical ambiguity][PL] occurs when a word has several possible meanings,
resulting in a sentence having multiple possible interpretations. In the
sentence:

> I like writing.

...it's unclear whether or not the author is referring to the act of writing
(the verb) or the result of writing (the noun). The best way to deal with
lexical ambiguity is to use a word that does not have multiple meanings, or
to rephrase the sentence such that the word now has only one possible meaning,
for example:

> I like writing stories.

In this case, however, we've narrowed the statement to referring to the act of
writing stories, so it's not a perfect match (though it may be what the author
intended). If we wanted to remain general, we would have to say:

> An activity I like is writing.

## Syntactic Ambiguity

[Syntactic ambiguity][SA] occurs when a sequence of words can be given more
than one grammatical structure. For example, consider the sentence:

> The police shot the rioters with guns

This can be interpreted as the rioters with guns were shot by the police, or
that the rioters were shot by police with guns; the grammatical structure of
the sentence is ambiguous.

## Semantic Ambiguity

[Semantic ambiguity][SM] occurs when a sentence has more than one way of being
interpreted within a context. Consider the sentence:

> Every student thinks she is a genius.

Without enough context, this sentence contains multiple possible
interpretations. Does each student in the class think they are a genius? Does
every student in the class think a particular girl in the class is a genius?
Does every student in the class think the female teacher is a genius? Without
sufficient context this sentence is easily prone to misinterpretation.

# Writing Unambiguously

A key part of writing unambiguous sentences is to be able to recognize an
ambiguous phrase as you write it. In the section on [detecting
ambiguity](#detecting_ambiguity), three core types of ambiguity were
identified: lexical, syntactic, and semantic. We're going to go over some
common examples, identifying the ambiguity in each and fixing them.

## "Only"

The qualifier "only" is a common source of ambiguity. Consider the following
example:

> The background task only marks jobs which have failed.

Here, the idiosyncrasies of the typical English speaker come to light. This
sentence is usually interpreted as saying that the jobs which have failed are
marked by the background task; the "only" is qualifying the noun "jobs".

However, in actuality the "only" is technically qualifying the word immediately
following it: the verb "marks", which gives the sentence a very different
meaning---namely that the background task only "marks" jobs, it does not
re-enqueue or perform any other sort of action for failed jobs, which is
probably not the intended interpretation.

This is an example of syntactic ambiguity, since there are two grammatical
structures we can assign to the sentence, resulting in different
interpretations. The best way to avoid ambiguity when using the "only"
qualifier is to place the "only" before the noun you're trying to qualify, and
not the verb. For our example above, we would change it to:

> The background task marks only jobs which have failed.

Writing in this way makes it sound odd as it doesn't match the ambiguous form
English speakers typically use, but it's unambiguous as to which word the
"only" is qualifying, and thus should be preferred in technical documentation.

## "All"

The qualifier "all" also lends itself to being responsible for ambiguous
sentences, like the following:

> All users have a unique identifier.

Here, there are two interpretations:

* All users share a common unique identifier
* Each user has their own unique identifier

The latter is probably the one that was intended, and would probably be the
interpretation most engineers have, but that's mostly because of their domain
experience in building systems (typically, in any system with users, each user
has an ID to uniquely identify them). In a situation where this domain
experience doesn't exist, this ambiguous phrasing is best avoided.

This is an example of semantic ambiguity, since different interpretations are
possible due to lack of context. The best approach for dealing with them is to
avoid using "all" entirely, and use "each" instead (in the case where you are
trying to describe a property for all objects).

## Plurals

Another interesting source of ambiguity can occur with plurals. For example,
consider the following two sentences:

> Users create over 10 posts per day.

> Users create over 10,000,000 posts per day.

Notice how the two sentences are exactly the same except for the actual number?
Yet the message of each is different. In the first sentence, since the number
is low we assume it is referring to the number of posts each user creates. In
the second sentence, 10,000,000 posts is obviously way too many for a single
person to create in a day, so it must be referring to the entire cohort in this
context.

This is another example of semantic ambiguity. This can be solved by
being more explicit in sentences where plurals are used. The first
sentence would become:

> Each user creates over 10 posts per day.

The second sentence would be better written as:

> Users collectively create over 10,000,000 posts per day.

## Synonyms

In English class, we were often taught to use synonyms to describe the same
word; this helped make our writing more interesting and varied.

Unfortunately, when it comes to technical specifications, using a synonym $Y$
to describe a word $X$ can be a source of confusion. For example:

> The background task constructs a list of words for use by the game engine.
> The process then uses the list of words when creating anagrams for the user.

In this situation, it's unclear whether "process" refers to the previously
mentioned background task, or a distinct process running the game. This form of
lexical ambiguity can be avoided by always referring to an object in the system
the same way each time, so that there is no possible misinterpretation.

# Writing in a Formal Language

The final approach to avoiding ambiguity is to write in a formal language.
Seeing as a large portion of ambiguities are caused by a lack of context, if we
move towards a [context-free language][CF] this can be avoided, as context-free
languages don't suffer from these perils.

When it comes to specification writing, a nice approach is to use the software
tests as a sort of specification for the software's behaviour.  This is known
as [Behaviour-Driven Developement][BDD], and is seeing a lot of use in in
[modern development processes][RS].

However, in most cases it's just easier to read a natural language, and so the
practical approach is to write specifications keeping the pitfalls of ambiguity
in mind.

[NL]: http://en.wikipedia.org/wiki/Natural_language
[AB]: http://en.wikipedia.org/wiki/Ambiguity
[PL]: http://en.wikipedia.org/wiki/Polysemy
[SA]: http://en.wikipedia.org/wiki/Syntactic_ambiguity
[SM]: http://en.wikipedia.org/wiki/Syntactic_ambiguity#Syntactic_and_semantic_ambiguity
[CF]: http://en.wikipedia.org/wiki/Context-free_language
[BDD]: http://en.wikipedia.org/wiki/Behavior-driven_development
[RS]: http://rspec.info/
