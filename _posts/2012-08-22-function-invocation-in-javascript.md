---
layout: post
title: Function Invocation in JavaScript
tags:
- JavaScript
---
I've recently made it a personal goal of mine to achieve a better understanding
of JavaScript. It's not that I'm terrible at using the language; I've been
developing with it using a variety of tools and frameworks like [jQuery][JQ]
and [Backbone.js][BB] for almost a year now, and have learned plenty in that
time.

The problem is that I don't feel like I _know_ the language. I'd be pretty
useless in any sort of philosophical discussion about it, and I certainly
wouldn't want to be on any committee deciding what is going into the
specification for the next edition of [ECMAScript][ECMA]. However, I'd like to
change that.

The following is the first of hopefully many informative pieces on the
JavaScript language. It may be pedantic at times, but my goal is to understand
these gritty details so that I can appreciate the language both top-down
and bottom-up.

# First-Class Functions

Functions are objects in JavaScript, and as such are values. This is weird
coming from languages like Java, but the concept quickly starts to make a lot
of sense. Being able to pass around a function as a callback is one obvious
example where this can be very powerful, as we don't have to construct one-off
objects with methods defined.

The one thing differentiating functions from objects is that they can be
invoked.

# What's `this`?

When a function is invoked, it has access to three things:

* Variables which are within its lexical scope (via a closure)
* Arguments passed to it via its parameters
* A variable `this`

How the `this` variable is initialized depends on how the function was invoked.

## Function Invocation

Any standalone function which is invoked as-is (i.e. it is not invoked as the
property of an object) will have `this` initialized to the global object (the
browser `window`, in this case).

{% highlight javascript %}
var echoThis = function () {
  console.log(this);
}

echoThis(); // Displays global `window` object
{% endhighlight %}

As [Crockford][CF] points out in [JavaScript: The Good Parts][JSGP], it would
have been much smarter if the language set `this` to be the whatever the value
of `this` was in its outer context. This way you could have helper functions
within your functions that could access the same `this` as the function they're
contained within, in theory making them more useful.

Luckily, an easy workaround is to just assign a different variable name to
`this` and access it within the helper function.

{% highlight javascript %}
var outer = function() {
  var that = this;
  var inner = function() {
    console.log(that);
  }
  inner();
}
{% endhighlight %}

## Method Invocation

Any time a function that is a property of an object is invoked, `this` will be
initialized to the object in question. This makes sense for most programmers
who come from classical object-oriented languages, and is thus perhaps the most
intuitive.

{% highlight javascript %}
var echoThis = function () {
  console.log(this);
}

var obj = { echoThis: echoThis };
obj.echoThis(); // Displays `obj`
{% endhighlight %}

Notice that while the function was originally declared as a standalone function,
since we made a property of `obj` a reference to that `echoThis` and invoked it
from that property, `this` was initialized to `obj`.

## Constructor Invocation

In order to appeal to programmers coming from classical languages, JavaScript
awkwardly included the `new` keyword in order to make object creation seem a
little more familiar. Unfortunately, it clouds JavaScript's [prototypal][PROTO]
nature, and ends up being a somewhat odd hack in the language.

Functions invoked via `new` result in the creation of an object with its
prototype linked to the function's prototype, and `this` bound to the newly
created object. This allows you to do something like the following:

{% highlight javascript %}
var Animal = function(matingCall) {
  this.matingCall = matingCall;
}

Animal.prototype.findMate = function() {
  console.log(this.matingCall);
}

var joey = new Animal("How you doin?");
joey.findMate(); // How you doin?
{% endhighlight %}

## Apply Invocation

Despite all the different kinds of invocations outlined above, there still come
times where you may want to explicitly assign a value to a functions `this`
variable. JavaScript's `Function` object has an `apply` method which allows you
to specify a binding context for `this` and an array of arguments to pass to
the function (the term [applying][FA] a function to its arguments comes from the
mathematical definition of a function).

{% highlight javascript %}
var monkey = function() {
  console.log('ooh ooh');
}

var invokeSpirit = function() {
  this();
}

invokeSpirit.apply(monkey); // ooh ooh
{% endhighlight %}

# Suddenly It All Makes Sense

Before learning the four different kinds of function invocation, the `this`
variable came across as a strange and seemingly unpredictable entity. After
learning them, however, I find myself using [`bind`][BND] less often, as I used
to use it at times when it wasn't necessary. It's another example of how
understanding your tools deeply has a tremendous impact on how you use them.


[JQ]: http://jquery.com "jQuery JavaScript Library"
[BB]: http://backbonejs.org/ "Backbone.js"
[ECMA]: http://www.ecmascript.org/docs.php "ECMAScript Documentation"
[CF]: http://en.wikipedia.org/wiki/Douglas_Crockford "Douglas Crockford"
[JSGP]: http://shop.oreilly.com/product/9780596517748.do "JavaScript: The Good Parts"
[PROTO]: http://en.wikipedia.org/wiki/Prototype-based_programming "Prototype-based Languages"
[FA]: http://en.wikipedia.org/wiki/Function_application
[BND]: https://developer.mozilla.org/en-US/docs/JavaScript/Reference/Global_Objects/Function/bind
