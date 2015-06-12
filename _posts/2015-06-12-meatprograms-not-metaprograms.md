---
layout: post
title: "Meatprogramming, not Metaprogramming"
author: "Samuel A. Falvo II"
---

Let me tell you a story of something that happened at work recently.

I've been put in charge of contributing a plugin to a reasonably popular systems integration mocking tool.
It offers support for so-called "control plane" APIs (a topic for another day),
which enables system/integration tests to control how this mocking tool behaves at run-time.
This tool is written in Python, a programming language I've been using since version 1.4.
This tool is written by many incredibly intelligent software engineers, all of whom I respect greatly.

Contributing my plug-in ought to be relatively easy to figure out.
I mean, I've been using Python since version 1.4 was a thing.
I know object oriented programming and decomposition techniques.
I'm aware of many different kinds of patterns and anti-patterns.

Yet, I can't even write one line of productive code for my plugin without severe code-coddling from my peers.

While discussing this with some other contributors,
it was concluded that the probem was inadequate documentation.
Now, people who know me know that I'm a huge proponent for literate programming,
so I'm hardly one to impede the documentation efforts of code authors.
However, and perhaps for the first time in history,
I think this is merely addressing the symptom, not the cause, of the problem.

I think the real cause of my problem is more fundamental:
an accidental regression from [structured programming](http://en.wikipedia.org/wiki/Structured_programming).


## Structured Programming

Structured programming came about to solve three different kinds of problems in computer science:
code clarity,
software quality,
and, improved productivity.

### Clarity

Structured programming improves clarity by establishing rules about how to read and write, and thus how to think about, code.
Replacing a chunk of code with a "black box",
typically but not necessarily always in the form of a subroutine,
enables the code reader to gloss over details irrelevant to understanding the code at hand.
Before block structure and, more formally, lexical scoping,
code sprawled all over the place (so-called [spaghetti code](http://en.wikipedia.org/wiki/Spaghetti_code)),
impeding a coder's understanding of the control flow to the point of submission. 

### Quality

Thanks to block structure and the principle of [Single Entry, Single Exit](http://anthonysteele.co.uk/the-single-exit-point-law),
a coder could actually *prove* correctness by treating subordinate code, already having been proved unto themselves, axiomatically.
When this principle is applied to the *design* process, we know it as [Stepwise Refinement](http://www.sqa.org.uk/e-learning/Planning02CD/page_14.htm),
and it's actually the same technique I'm using to develop the [Kestrel-3's firmware](http://sam-falvo.github.io/kestrel/2015/05/17/stepwise-refinement-of-forth-interpreter/).

Now, it turns out that the SESE principle is a bit too restrictive in practice;
today, we know that SEME (Single Entry, Multiple Exit) works better.
How do we know this?
By using SESE itself and formally reasoning about code written in both SESE and SEME styles, we can easily establish an equivalence.
In fact, if you look under the hood, you'll find that most compilers *compile* SEME code into SESE code anyway,
so you still get the benefits the SESE principle provides.

See?    SESE works.

### Productivity

Structured programming aims to reduce coding effort, and thus time expended,
by cataloging a *small* number of highly orthogonal [design patterns](http://en.wikipedia.org/wiki/Software_design_pattern) which frequently appeared in high-quality software.
These break down into three broad categories:
**Sequence** (ordered collection of statements or their equivalents, such as subroutines),
**Selection** or **Alternation** (execute one of *n* different code paths based on some criteria), and,
**Iteration** (execute the same code path *n* times, or until a condition becomes satisfied).
These patterns are so well understood these days that all major programming languages today supports explicit syntax for each of these statements.
It's hard to believe, but even as late as 1984, many languages actually lacked a `while` construct or the ability to invoke a subroutine without an explicit `CALL` keyword.

### Compound Effects and Hierarchical Design

By combining block structure,
introducing rules allowing correctness proofs (even if informally), thus allowing one to verify their own understanding of the code,
and by offering a standard catalog of common design patterns,
the gains *compounded*, allowing the programmer development speeds far faster *and* with fewer errors than with competing methodologies of the time.

Applying all three concepts with skill leads to software which has a natural hierarchy to it.
Higher level code tends to mediate or coordinate lower-level code.
Lower-level code tends to consist mostly of operations and/or data accessors.
I encourage readers to look into [Ralf Westphal's IODA Architecture](http://geekswithblogs.net/theArchitectsNapkin/archive/2015/04/29/the-ioda-architecture.aspx) if you want to know more.
**Even if your code doesn't actually run in a predominantly sequential fashion, your code can still benefit from this architecture.**
Indeed, browsing around Ralf's website will reveal illustrations and examples of event-driven applications written IODA-style.
The code is a joy to read.

## The Problem with Dynamic Languages and Metaprogramming

I'm a Forth programmer.  Metaprogramming, as it is with Lisp, is in my blood.
However, there's little disagreement from me that metaprogramming can be easily abused.
In Forth, it comes in the form of `IMMEDIATE` words.    In Lisp, it comes in the form of macros.
But, in Python and other object-oriented, dynamically-typed programming languages, it comes in the form of *functions.*

Forth is also the ultimate in dynamically-typed languages: it has *no* types of any kind, except the machine word.
Thus, I receive no benefit from the compiler or interpreter when I make a type-related error.
Which I make quite frequently;
certainly, enough for me to swear in public that I'd never use Forth again.
(And, yet, I always go back to Forth.)

Speaking as someone fluent in Forth, Python, Ruby, and other dynamic languages,
I mention this only because I feel that dynamically-typed languages *encourage* the use of metaprogramming *too much.*
With a more restrictive environment that focuses only on the essentials,
you may find your desire for cleverness increases, but begrudgingly,
you will write more maintainable code.

For instance, while working on the Kestrel-2's system software,
I frequently lamented [not having access to a return stack](http://sam-falvo.github.io/kestrel/2013/12/08/programming-without-lambdas/).
I used a dialect of Forth that simply didn't support application-custom immediate words, `CREATE`, `>R` or `R>`, or other metaprogramming tools,
mainly because it was a target compiler, and because the underlying hardware just didn't support that functionality.
You know what?
I can still read and support the code today, many years later.
The code compiles to this day, without compile-time or run-time error.

So what's my beef, then?
Although patently contrived,
here's an example that's actually inspired by code in that mocking tool I talked about earlier.

    def create_comparator_class(low_bound, high_bound):
        class Comparator(object):
            """docstring goes here."""
            def compare(c_self):
                """another docstring here."""
                if c_self.get_left() < c_self.get_right():
                    return -1
                elif c_self.get_left() == c_self.get_right():
                    return 0
                else
                    return 1

        def get_lower_bound(c_self):
            return low_bound

        def get_high_bound(c_self):
            return high_bound

        setattr(Comparator, "get_left", get_lower_bound)
        setattr(Comparator, "get_right", get_high_bound)

        return Comparator

With this code, we can now create any number of comparator classes simply by mentioning a set of parameters:

    CompareLetters = create_comparator_class('a', 'z')
    CompareNumbers = create_comparator_class(1, 10)

It should be readily apparent to the reader is that we're *effectively* synthesizing code on-the-fly.
In other words, we're exploiting [self-modifying code](http://en.wikipedia.org/wiki/Self-modifying_code), in spirit if not in fact.

There are several problems with this technique:

1.  How do you document the `create_comparator_class` function?  I mean, really think about it.  Python docstring formatting conventions are not well equipped to adequately document, *to the same level as a statically declared class docstring*, what the resulting classes do, what their methods are capable of, what pre- and post-conditions exist, and so forth.  In fact, I'm willing to bet you that if you use code like this in your project, it probably won't have a docstring longer than 15 lines.  The example above, as simplistic as it is, already is 5 lines longer than that, and we didn't even consider how to dynamically generate docstrings for the `get_left` and `get_right` methods.

2.  It increases cyclomatic complexity, impeding the maintainer's ability to understand what's going on, and more importantly, *why*.  Some projects I've worked on actually have a continuous-integration gate on cyclomatic complexity because of how problematic things can get.

3.  I don't say this often; but I'll say it here.  This is one case where Python's indent-based blocking is a real disadvantage.    While maintaining Python code like this, you have to read and comprehend the entire outer definition *and all inner definitions* before you can even *begin* to consider, "Hey, are these inner definitions indented properly?".    Are you *sure* all your `def`s are properly aligned?    Without static checking, you cannot know unless the code is actually executed and all code paths have been exercised.  I hope you have a good code coverage tool!  If you didn't see the `setattr` functions at the end of the definition, or if they happened to be buried in a lower-level function somwhere out of sight, would you have complained about the methods not having the right indent level?  These are not hypothetical concerns; they happen in real projects, in real time, every day.

4.  Up to an inflection point, it's more code than you need to write.  Software defect rates are known to be correlated with total lines of code, regardless of programming language used.  Therefore, why choose to write more lines when you can write fewer?

It turns out Python has a perfectly serviceable method of creating new classes as we need them in the source code:

    class ComparatorBase(object):
        def compare(self):
            """Return the result of comparing two values."""
            return self.a < self.b

    class C1(ComparatorBase):
        def __init__(self):
            self.a = 1
            self.b = 10

    class C2(ComparatorBase):
        def __init__(self):
            self.a = 'a'
            self.b = 'z'

That's 12 lines of code, as compared to 19, despite the more verbose attribute assignment.
Indeed, with the original approach,
we don't see a savings in total lines of code until we instantiate more than four subclasses,
and even then, we can introduce a "maker" function that is still substantially simpler:

    def create_comparator_class(low_bound, high_bound):
        class Cx(Comparator):
            def __init__(self):
                self.a = low_bound
                self.b = high_bound
        return Cx

When errors are measured in defects per 1000-lines-of-code, this small difference is insignificant.
But, when you approach 2000 or more lines of code in your project, suddenly it becomes measurable, even to an individual.

We gain other benefits from this simplification as well.

1.  The code flows are patently obvious to the reader.
This *reduces your documentation burden*, so that
you don't *have* to document how things work at such minute levels that you're basically documenting how the language works.

2.  It's provably correct *by inspection*.
But, if you don't trust your ability to inspect code,
or you inherently distrust code written by 3rd party teams,
it's also fully compatible with, e.g., Python 3's ability to support tooling around optional static typing.

3.  It's substantially easier to document.  You can use normal Python-style docstring techniques to document the base class, and refer to it in `create_comparator_class`'s docstring as needed.

Those are important benefits exploited simply by using *static* program structures in favor of *dynamic* structures.

Alas, in the mocking tool, we *can't* use statically structured code like my illustration above,
because of a problem introduced by yet another metaprogramming facility: **decorators.**

Unlike Java's decorators, Python's decorators are intended to be compile-time functions of other Python constructs
(usually classes and functions themselves), and
this particular application uses them to specify URLs for RESTful endpoint dispatching.
It follows that we can't easily parameterize a RESTful URL without somehow digging into the internals of the REST framework's guts.
**This utterly defeats the benefits and purpose of modular programming en toto.**

Had the REST framework been architected with more static constructs in mind,
the decorator would have been written in terms of a more general-purpose mechanism for adding URL routes,
and this whole mess could have been avoided.
We could have used a much simpler interface for plugin writers,
such as using a *dictionary* to map URL to handler method or class,
arguably far more obvious to the code reader as it provides better separation of concerns.

Alas, they didn't, and it's not, and so the decorator is the *only* (documented) method of adding a route to the web-app class instance.

## You Suck.

At this point, you're probably thinking that I'm just not that good of a Python programmer.
Or, more generally, not that good at higher-level thinking in general.
I hear you.  And, you're probably right.
I'll be the first to admit that my aptitude and proclivities align towards lower-level, simpler programming.
Honestly, though, feedback from my fellow maintainers suggest that I'm not that bad as engineers go.
And, yes, I speak with my peers frequently about my self-perceived deficiencies.

So, at the end of the day,
if you're reading this and thinking that I'm just not up to snuff or somehow "not good enough" to hack "real" Python code,
then you're falling prey to the [No True Scotsman](http://rationalwiki.org/wiki/No_True_Scotsman) fallacy.
Indeed, I can reverse the argument back on to you:
if you can't write semantically clean, easily maintainable code without resorting to cleverness,
you're not that experienced an engineer yourself.
But, then, we'd just end up in a flame war that goes nowhere, wouldn't we?

## Conclusion and My Plea To You

The code is already written; I have to bite the bullet and deal with it.
But, I'd like to plea with you, the reader, for mercy when writing new code.

If this blog post serves any purpose at all,
it is hopefully to get you, the reader, the high-level, dynamically-typed
programming language coder, to think twice every time you *even consider* a metaprogramming solution.
This includes macros in Lisp, immediate words in Forth, and decorators in Python.

I'm not alone in this.  [If you do a Google search for "thoughts on metaprogramming"](https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=thoughts%20on%20metaprogramming),
you'll find a litany of webpages describing how people just *hate* metaprogrammed solutions, for one reason or another.

Programming languages serve two audiences: humans, and computers.
You've already mastered instructing the computer.
Now you need to master how to write code to support your fellow human being.
You need to write [meatprograms](http://therealadam.com/2011/12/09/why-metaprogram-when-you-can-program/), not metaprograms.
Your fellow engineers will thank you, and most importantly,
your employer will be thankful for not having to spend loads of cash on engineers trying to reverse-engineer some obscure bit of cleverness when they could be making progress instead.

