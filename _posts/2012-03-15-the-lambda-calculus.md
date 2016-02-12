---
layout: post
title: Understanding the Lambda Calculus
tags:
- computer science
- lambda calculus
---
I've recently been learning [functional programming][FP] from the ground up. My
previous attempts---while not complete failures---typically resulted in me
learning how to program in a functional language, but I didn't feel like I was
understanding the core philosophy of the paradigm.

Obviously, it's not always necessary to have a deep understanding of theory
before being able to successfully apply a concept. However, in the case of
functional programming, it really is centered around a mathematical model of
computation (and a quite beautiful one, at that). I would be so bold as to say
that any self-respecting hacker should understand these fundamentals---they
really do change how you think about computation, and for the better.

With that said, the following is a distillation of what I think are the
building blocks behind one of the origins of functional programming: the lambda
calculus. My hope is that this will provide enough of a baseline to encourage
further exploration in understanding this important development in computer
science.

# A Simple Model of Computation

The [lambda calculus][LC] is a formal system for expressing computation. The
term _calculus_ is well-chosen, as it is meant to indicate that the system is
very small and simple (the original Latin meaning of calculus is [small
pebble][CL]).

This simplicity at first seems limiting, but it is for good reason. It allows
us to easily prove properties of the system, and these properties will still be
valid as we add features to the system to make it more like the programming
languages we know and love.

So how simple are we talking? Well, here's a full grammar for all lambda
calculus expressions, in [Backus-Naur form][BNF] (don't worry if you aren't
familiar with the notation---I'll explain in detail below):

<div title="Grammar for lambda calculus expressions">
$$
\begin{aligned}
\langle expression \rangle &amp; ::= \langle variable \rangle \\
                           &amp; | \hspace{7mm}  \langle abstraction \rangle \\
                           &amp; | \hspace{7mm}  \langle application \rangle \\
\langle abstraction \rangle &amp; ::= \lambda \langle variable \rangle .
                                      \langle expression \rangle \\
\langle application \rangle &amp; ::= \langle expression \rangle
                                      \langle expression \rangle
\end{aligned}
$$
</div>

If you've ever seen the BNF grammar for any popular programming
language, you'll immediately realize from the grammar above how simple
lambda calculus expressions are. In a nutshell, an expression can be one of
three things: a variable name, a function declaration (called an
_abstraction_), or a function invocation (called an _application_). The
simplest expression is therefore:

<div title="The simplest lambda calculus expression">
$$ x $$
</div>

What is $x$? It's just a variable, free to take on any value we choose (you can
think of it as an input to your program).

To declare functions (also referred to as abstractions), we write a &lambda;
followed by a variable name to declare the parameter of the function, and
follow it by another expression representing the _body_ of the function.
Thus a simple function could be:

<div title="The identity function">
$$ \lambda x . x $$
</div>

This is an example of the _identity_ function---it takes a parameter $x$ and
returns whatever argument was given for $x$. For example, applying the identity
function to the value $y$, we get the original value $y$:

<div title="Applying the identity function to the value y">
$$ (\lambda x . x) y \rightarrow y $$
</div>

We just saw an example of the final kind of expression: an application. In an
application, the abstraction ($\lambda x . x$) is applied to the argument $y$.
We can view the process of application as substituting $y$ for $x$ everywhere
that $x$ appears in the body of the abstraction, removing the $\lambda$ and
variable name while we're at it. The technical terms for the left-side and
right-side are _rator_ (from _operator_) and _rand_ (from _operand_),
respectively.

<aside title="A Note on Brackets">
<p>
Throughout this article I'll try to maximize my use of brackets so as to
eliminate any ambiguity in expressions. However, there is an accepted practice
for minimizing the number of brackets while keeping expressions unambiguous.
This practice consists of the two following rules:
</p>

<ul>
  <li>
    Function application is left-associative: $xyz$ means $((xy) z)$.
  </li>
  <li>
    Abstractions extend as far right as possible:
    $\lambda x.yz$ means $\lambda x.(yz)$, not $(\lambda x.y)z$.
  </li>
</ul>
</aside>

One quick note that should be mentioned is that there is no requirement for a
variable of an abstraction to appear in the body of the abstraction. Therefore,
the following is also a valid abstraction:

<div title="The constant function">
$$ \lambda x . y $$
</div>

This is an example of a _constant_ function. It essentially ignores its
argument and always returns the same value $y$, regardless of the value
substituted for $x$.

# Functions Are Values

At first glance, it would seem that the lone $x$ expression and the identity
function $\lambda x.x$ mentioned earlier are the same thing. For example, $x$
can take on any value in the former, and any value can be substituted for $x$
in the latter.

There is, however, a quite important distinction, which is why these two
expressions are indeed far from the same. In the second expression, the
abstraction itself is the value of the expression. Since it is not applied to
any argument, no value is substituted for $x$, and so it remains as $\lambda
x.x$. It cannot be simplified further, because that changes the meaning of the
expression.

The concept of treating functions as values is one of the most important traits
of functional programming. It allows us to create functions that can take other
functions as arguments, as well as return functions as results. This flexibility
gives the lambda calculus an incredible amount of expressiveness. A lot of modern
programming languages have included this ability in their own design for this
very reason. JavaScript, for example, frequently uses the passing of functions
as values in order to supply custom callback code for asynchronous requests.

# Functions of Multiple Variables

It may seem odd that functions in the lambda calculus can only be declared with
a single parameter.  However, we can see that this does not sacrifice any
amount of expressivity. We can define a function which takes more than one
parameter to simply be a function that returns a function that takes a
parameter. For example, consider the following function:

<div title="A curried function of two variables">
$$ \lambda x . \lambda y . (x y) $$
</div>

The expression above takes a paramater $x$ and returns a function that takes a
parameter $y$, which in turn returns the result of applying $x$ to $y$.
Therefore, in order to apply the function to two arguments, we would first have
to apply it to a value to substitute for $x$, and then apply the result of that
to a value to substitute for $y$, i.e.:

<div title="A curried function of two variables with arguments applied">
$$ ((\lambda x . \lambda y . (x y)) (\lambda z. z)) w $$
</div>

Here, we applied the value $\lambda z.z$ to the function first, then applied
the result of that (notice the placement of the brackets) to the value $w$. If
we evaluated this expression, it would reduce to $w$, since the function
applies the first argument (the identity function) to the second, resulting in
the second argument $w$ being returned.

In general, a function of $n$ variables ($n > 1$) will be represented by a
function of one variable that produces a function of $n - 1$ variables.
Functions of this kind are called [curried][CF] functions (named after
[Haskell Curry][HC]), and have some nice properties that we won't get into now.

# Free and Bound Variables

We've talked about defining functions and applying them to arguments, but we
haven't formalized how the variables in the body of a function are substituted
during an application. In order to do so, we need to talk about the concept of
_free_ and _bound_ variables in a lambda calculus expression.

In the expression consisting of the lone variable $x$, $x$ is free to take on
any value, thus $x$ is considered a free variable. However, in the body of the
abstraction $\lambda x.x$, we say that $x$ is bound by the _binding occurrence_
of $x$ as the abstraction's parameter.

It's still possible for the body of an abstraction to contain free variables, so
long as they are not bound by any binding occurrences in the containing abstraction
definitions. For example, $y$ is free in $\lambda x.y$.

We can define the set of free variables ($FV$) and bound variables
($BV$) of a lambda calculus expression as follows:

<div title="Formal definitions of free and bound variables for expressions">
$$
\begin{align*}
FV[x]           &amp; = \{x\} \\
FV[\lambda x.E] &amp; = FV[E] - \{x\} \\
FV[MN]          &amp; = FV[M] \cup FV[N] \\
\\
BV[x]           &amp; = \emptyset \\
BV[\lambda x.E] &amp; = BV[E] \cup \{x\} \\
BV[MN]          &amp; = BV[M] \cup BV[N]
\end{align*}
$$
</div>

These definitions are relatively intuitive, given what we've currently
discussed about free and bound variables. The main idea to note is that
abstractions are responsible for introducing bound variables and removing free
variables.

Given these definitions of free and bound variables, we can now start reusing
the same variable name in an expression (this has been avoided up until now).
For example, $x$ is both free and bound in the following expression:

<div title="Expression where x is both free and bound">
$$ (\lambda x.x) x $$
</div>

Why is this the case? The rightmost occurrence of $x$ is free because it is not
contained in the body of any abstraction, thus it can take on any value. The
middle occurrence of $x$, however, is bound by the binding occurrence of the
leftmost occurrence of $x$. This value of $x$ cannot take any value; it must
take the value that is substituted for the bound occurrences of $x$ in the
body.

# Substitution

When we apply an abstraction to its argument, we replace all occurrences of
the parameter of the abstraction with the argument expression. We use the
following notation to represent this process:

<div title="Notation for substituation">
$$ (\lambda x.B) E = [x \mapsto E]B $$
</div>

This notation simply means that all free occurrences of $x$ in $B$ are replaced
with $E$.

However, we have to be careful when performing this substitution, since if we
blindly replace the variable we may end up causing a variable in the expression
we're replacing it with to be unnecessarily captured by a binding occurrence of
an outer abstraction. For example, $\[x \mapsto y\](\lambda y.xy)$ should not
become $\lambda y.yy$, as that changes the meaning of the expression. When we
substituted $y$ for $x$, $y$ was free, but after the substitution $y$ became
bound.

In order to prevent this accidental capture of free variables, we use the
following recursive definitions when performing substitution:

<div title="Recursive definition of substitution">
$$
\begin{aligned}
\lbrack x \mapsto E \rbrack x &amp; = E \\
\lbrack x \mapsto E \rbrack y &amp; = y \\
\lbrack x \mapsto E \rbrack (MN) &amp; =
  (\lbrack x \mapsto E \rbrack M)(\lbrack x \mapsto E \rbrack N) \\
\lbrack x \mapsto E \rbrack (\lambda x.P) &amp; = \lambda x.P \\
\lbrack x \mapsto E \rbrack (\lambda y.P) &amp; =
  \lambda y.(\lbrack x \mapsto E \rbrack P), y \notin FV \lbrack E \rbrack  \\
\lbrack x \mapsto E \rbrack (\lambda y.P) &amp; =
  \lambda z.(\lbrack x \mapsto E \rbrack \lbrack y \mapsto z \rbrack P),
  y \in FV \lbrack E \rbrack , z \text{ is a fresh variable} \\
\end{aligned}
$$
</div>

The last definition is what prevents us from accidentally capturing free
variables. If the parameter of the abstraction occurs free in the expression
we're substituting, we need to rename the variable with a fresh variable so that
no accidental capture takes place.

# Equivalent Expressions

In the previous example we learned that $(\lambda x.x)x$ contains both free and
bound variables, despite these variables having the same name. This would
perhaps make more sense if we were trying to find the free and bound variables
of $(\lambda y.y)x$, as it becomes clearer that $x$ is free and $y$ is bound
with respect to the abstraction it is contained within. In a way, the two
expressions $(\lambda x.x)x$ and $(\lambda y.y)x$---while containing different
variable names---were equivalent.

Formally, we call two expressions that are similar in this way
_$\alpha$-equivalent_, written as:

<div title="Notation for &alpha;-equivalence">
$$
(\lambda x.x)x =_{\alpha} (\lambda y.y)x
$$
</div>

To define $\alpha$-equivalence, we need to be careful. The expression $x$
and the expression $y$ on their own are not $\alpha$-equivalent to each
other, as they are two free variables which individually could take on any
value.

<div title="Two differently named variables are not &alpha;-equivalent">
$$
x \ne_{\alpha} y
$$
</div>

However, the abstractions $\lambda x.x$ and $\lambda y.y$ _are_
$\alpha$-equivalent.

<div title="Two abstractions that are &alpha;-equivalent">
$$
\lambda x.x =_{\alpha} \lambda y.y
$$
</div>

From these examples we can see that $\alpha$-equivalence is simply a way of
formalizing when two lambda calculus expressions are semantically equivalent
(i.e. they compute the same thing). If we can rename bound variables so that
both expressions are exactly the same, while at the same time preserving the
semantics of the expression, those two expressions are $\alpha$-equivalent.

Formally, we can define $\alpha$-equivalence using the following recursive
definitions:

<div title="Definition of &alpha;-equivalence">
$$
\begin{aligned}
x &amp; =_{\alpha} x,&amp; \text{ (i.e. they are the same variable)} \\
x &amp; \ne_{\alpha} y,&amp; \text{ (i.e. they aren't the same variable)} \\
\lambda x.E_1 &amp; =_{\alpha} \lambda y.E_2,&amp;
  \text{if } \lbrack x \mapsto z \rbrack E_1 =_{\alpha} \lbrack y \mapsto z \rbrack E_2,
  \text{ where } z \text{ is a fresh variable} \\
E_1 E_2 &amp; =_{\alpha} E_3 E_4,&amp;
  \text{if } E_1 =_{\alpha} E_3, E_2 =_{\alpha} E_4
\end{aligned}
$$
</div>

The cases for variables and applications are pretty straightforward.
Abstractions require that we rename the variable name of each abstraction to the
same *fresh variable* (i.e. a newly-generated variable that is not used
anywhere else in the expression), and if the bodies of the abstractions with the newly
substituted variable are the same, then the two abstractions are the same.

# Computation

Now that we've gotten a sense for lambda calculus expressions, the next question
is: what exactly does computation in this model look like?

Substitution is a core part of the semantics of the lambda calculus. We know
that the expression $(\lambda x.yx)z$ should reduce to $yz$, since $z$ is
subtituted for $x$ in the abstraction. We call this substitution process
[$\beta$-reduction][BR], and denote it:

<div title="Notation for &beta;-reduction">
$$
(\lambda x.yx) z \rightarrow_{\beta} yz
$$
</div>

There may be more than one way to reduce an expression. For example, the
expression $(\lambda x.(\lambda y.x)w)z$ can be reduced in two different ways:

<div title="Two different reductions of the same expression">
$$
\begin{aligned}
(\lambda x.(\lambda y.x)w)z &amp; \rightarrow_{\beta} (\lambda y.z)w \\
(\lambda x.(\lambda y.x)w)z &amp; \rightarrow_{\beta} (\lambda x.x)z
\end{aligned}
$$
</div>

In the first example, we reduced the outermost application by substituting $z$
for all bound occurrences of $x$. In the second example, we reduced the
innermost application, substituting $w$ for all bound occurrences of $y$ (of
which there were none).


If an expression can be simplified in this manner, we say that expression is
$\beta$-reducible. If it can't, we say that the expression is in
[$\beta$-normal form][BNRF] (or just normal form).

We can now think of computation as simply repeatedly reducing an expression to
its normal form. However, does every expression have a normal form? Considering
that one can write a program with an infinite loop, it seems like not all
expressions should have a normal form (i.e. computation does not terminate for
all expressions), and indeed this is the case. The simplest example is the
$\Omega$ expression:

<div title="Simplest non-converging lambda calculus expression">
$$
(\lambda x.xx)(\lambda x.xx) \rightarrow_{\beta} (\lambda x.xx)(\lambda x.xx)
$$
</div>

In this case, we say that the expression _diverges_ under $\beta$-reduction.

One important result with regards to normal forms is the [Church-Rosser
theorem][CRT], which states that any lambda calculus expression has at most one
normal form. In other words, an expression will either diverge forever or will
eventually reduce to a normal form.

# Finding the Answer

A problem with arbitrarily reducing an expression is that while a normal form
may exist, we may perform a reduction which leads us down a path that results in
divergence. For example consider the following two reductions of the same
expression:

<div title="Example of an expression that both diverges and has a normal form">
$$
\begin{aligned}
(\lambda y.z) ((\lambda x.xx)(\lambda x.xx)) &amp; \rightarrow_{\beta}
(\lambda y.z) ((\lambda x.xx)(\lambda x.xx)) \\
(\lambda y.z) ((\lambda x.xx)(\lambda x.xx)) &amp; \rightarrow_{\beta} z
\end{aligned}
$$
</div>

In the first case, we reduced the innermost application,
$(\lambda x.xx)(\lambda x.xx)$, which reduced to the same value, and thus
resulted in no change. In the second case, we reduced the outermost application
of the abstraction $\lambda y.z$ to its argument, which simplified to the value
$z$, since $y$ is not bound anywhere in the body of the abstraction.

Therefore, it is important that we choose our $\beta$-reductions carefully.
Luckily, reducing the leftmost, outermost application will always find a normal
form, if it exists. This [reduction strategy][RS] is called Normal Order
Reduction (NOR). A series of normal order reductions are shown below:

<div title="Full example of normal order reduction">
$$
\begin{aligned}
((\lambda x.(\lambda y.x (\lambda z.yz)r))w)q &amp; \rightarrow_{\beta}
(\lambda y.w (\lambda z.yz)r)q \\ &amp; \rightarrow_{\beta}
w(\lambda z.qz)r \\ &amp; \rightarrow_{\beta}
w q r
\end{aligned}
$$
</div>

While probably the most intuitive of the reduction strategies, NOR is rarely
used in actual programming languages, because it requires us to rewrite
abstractions during the process of computation. This can get expensive, so NOR
tends to be avoided for practical use.

Normal Order Evaluation (NOE) is another reduction strategy that is very
similar to NOR, except it forbids reduction within the body of abstractions. It
is also known as [call by name][CBN], as arguments to a function are not
evaluated beforehand, but are rather substituted directly into the body and
left to be evaluated when the entire function is evaluated later. The Haskell
language uses a modified version of call by name, [call by need][CBNE], which
is where if the function argument is evaluated, its value is stored for
subsequent uses (resulting in less repeated computation if the parameter is
used multiple times in the abstraction).

Applicative Order Reduction (AOR) reduces the leftmost, innermost application,
and Applicative Order Evaluation (AOE) does not reduce applications in the body
of abstractions. AOE is also known as [call by value][CBV], and is used in
functional programming languages like [Scheme][SCH] and [ML][ML], as well as
most imperative programming languages.

Here's a concrete example of an expression where reduction by NOR, AOR, and AOE
all result in a different initial reduction, but whose repeated application
leads to the same normal form.  Reduction by NOR and NOE are the same in this
list, as by definition NOE will either be the same or will stop earlier due to
its restrictions on not simplifying applications within abstractions.

<div title="Examples of reductions using NOR, NOE, AOR, and AOE">
$$
\begin{aligned}
(\lambda x.(\lambda y.y)x)((\lambda w.w)z) &amp; \overset{NOR}{\rightarrow}
(\lambda y.y)((\lambda w.w)z) \\           &amp; \overset{NOR}{\rightarrow}
(\lambda w.w)z                \\           &amp; \overset{NOR}{\rightarrow}
z \\
(\lambda x.(\lambda y.y)x)((\lambda w.w)z) &amp; \overset{NOE}{\rightarrow}
(\lambda y.y)((\lambda w.w)z) \\           &amp; \overset{NOE}{\rightarrow}
(\lambda w.w)z \\                          &amp; \overset{NOE}{\rightarrow}
z \\
(\lambda x.(\lambda y.y)x)((\lambda w.w)z) &amp; \overset{AOR}{\rightarrow}
(\lambda x.x)((\lambda w.w)z) \\           &amp; \overset{AOR}{\rightarrow}
(\lambda x.x)z \\                          &amp; \overset{AOR}{\rightarrow}
z \\
(\lambda x.(\lambda y.y)x)((\lambda w.w)z) &amp; \overset{AOE}{\rightarrow}
(\lambda x.(\lambda y.y)x)z \\             &amp; \overset{AOE}{\rightarrow}
(\lambda y.y)z              \\             &amp; \overset{AOE}{\rightarrow}
z
\end{aligned}
$$
</div>

The main takeaway from all this is that different methods of computation can
result in the same final answer (normal form). Computation in different
programming languages varies based on the evaluation strategy used, and each
strategy has different advantages and disadvantages that these languages
utilize.

# How Do You Program in the Lambda Calculus?

If you've made it this far, you're probably wondering where all of this is
going. Given how apparently restrictive the lambda calculus is, how does one
actually write any non-trivial program with it? Where are the *if*
statements, the arrays, or even the ability to perform arithmetic?

Remember that the lambda calculus was designed to be incredibly small and
simple so that it was easy to prove various properties about it. In order to be
able to apply these properties to higher-level languages, we now have to add the
features of these languages by using the basic building blocks of the lambda
calculus.

Rest assured that being able to add arithmetic and other high-level language
features is possible. [Church encoding][CE] provides a means of including
higher-level operators and data into the lambda calculus.

To give you a taste of what is involved with these additions, we'll show how to
add the concept of *true* and *false* along with *if* statements to the lambda
calculus. This specific Church encoding technique is known as
[Church booleans][CB].

To keep us from repeating ourselves to much, we're going to abuse the notation
slightly and introduce abbreviations to make expressions easier to read. For
example, the identity function will be represented as:

<div title="Shorthand notation for the identity function">
$$
\textbf{id} = \lambda x.x
$$
</div>

To represent the Boolean *true* and *false* values, we consider how we could
design them so that they could be used by *if*. *if* statements can be thought
of as functions that take 3 parameters: a Boolean expression $B$ to evaluate,
an expression $T$ to return if $B$ is *true*, and an expression $F$ to return
if $B$ is *false*.

Consider then, the following definitions:

<div title="Defintions for true, false, and if">
$$
\begin{aligned}
&amp; \textbf{true} = \lambda x. \lambda y. x \\
&amp; \textbf{false} = \lambda x. \lambda y. y \\
&amp; \textbf{if } B \textbf { then } T \textbf{ else } F = B T F
\end{aligned}
$$
</div>

Here, we represent *true* as a function that returns the first of two
parameters, and *false* as a function that returns the second of two
parameters. This works beautifully with an *if*, as then we simply need to
apply the two expressions $T$ and $F$ from the *if* statement to the Boolean
expression $B$, and the definitions of *true* and *false* will ensure that the
correct expression is returned.

Using this same idea, we can also define *and*, *or*, and *not*.

<div title="Definitions of and, or, and not">
\begin{aligned}
&amp; A \textbf{ and } B = A \text{ } B \textbf{ false} \\
&amp; A \textbf{ or } B = A \textbf{ true } B \\
&amp; \textbf{not } A = A \textbf{ false } \textbf{true}
\end{aligned}
</div>

Representing *true* and *false* as functions becomes really
powerful, allowing us to define Boolean operators by simply applying the
arguments to those operators to each other.

# There's So Much More
We've really just scratched the surface of what is possible with the lambda
calculus, but hopefully it was deep enough that it's easy to see the power that
this simple model of computation contains, and also where various features of
functional programming languages came from.

If you are interested in learning more, the next step is to start adding types
to the calculus. So far we've been dealing with an untyped lambda calculus,
where no rules are in place to ensure that the right kind of arguments are
given to a function---we just assumed that anyone writing a program in this
model wouldn't make such a mistake. The [simply typed lambda calculus][STLC] is
the first step into dealing with types in a model of computation, and will
eventually lead you to other calculi such as the [polymorphic lambda
calculus][SF].

[FP]: http://en.wikipedia.org/wiki/Functional_programming
[LC]: http://en.wikipedia.org/wiki/Lambda_calculus
[CL]: http://en.wiktionary.org/wiki/calculus#Latin
[BNF]: http://en.wikipedia.org/wiki/Backus%E2%80%93Naur_Form
[CF]: http://en.wikipedia.org/wiki/Currying
[HC]: http://en.wikipedia.org/wiki/Haskell_Curry
[BR]: http://en.wikipedia.org/wiki/Lambda_calculus#Beta_reduction
[BNRF]: http://en.wikipedia.org/wiki/Beta_normal_form
[CRT]: http://en.wikipedia.org/wiki/Church%E2%80%93Rosser_theorem
[RS]: http://en.wikipedia.org/wiki/Lambda_calculus#Reduction_strategies
[CBN]: http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_name
[CBNE]: http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_need
[CBV]: http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_value
[SCH]: http://en.wikipedia.org/wiki/Scheme_%28programming_language%29
[ML]: http://en.wikipedia.org/wiki/ML_%28programming_language%29
[CE]: http://en.wikipedia.org/wiki/Church_encoding
[CB]: http://en.wikipedia.org/wiki/Church_encoding#Church_booleans
[STLC]: http://en.wikipedia.org/wiki/Simply_typed_lambda_calculus
[SF]: http://en.wikipedia.org/wiki/System_F
