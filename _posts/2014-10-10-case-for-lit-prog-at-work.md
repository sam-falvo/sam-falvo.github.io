---
layout: post
title: "A Case for Literate Programming at Work"
date: 2014-10-10 17:25:00
categories: literate programming knuth work software complexity
---

## Abstract

Recently, I suffered difficulties trying to utilize a testing tool at work.
The primary contributors were unavailable for questioning as a result of their other daily tasks, leaving me blocked on my own tasks.
If they used literate programming techniques when developing their product, I could have learned what I needed to know about it much faster.
The improved asynchrony of our respective teams implies greater productivity of the company as a whole.

## My Frustration

Several weeks ago,
I took on the task of writing a regression test suite for a project at the office.
At first, I was going to write them in Go, as I'm familiar with the language and its tools;
the built-in framework does exactly what we want, and
I could have built on top of another Go project's tests as a foundation.
However, management encouraged me to look into using a more widely accepted framework inside the product-QE group.
I said OK, as I couldn't see a good reason why not at the time.

However, it's taken me several weeks of effort to learn how to get this tool to invoke any tests I write.
The installation instructions were solid and well-written.
However, the documentation fails to provide any tutorial documentation to let me replicate such a simple task successfully.
(I intend on helping to fill this gap, of course.)
Fellow engineers in my office who have worked with previous versions of this tool quickly found the latest version behaved differently from their expectations as well.
I've sent e-mails to trusted coworkers on this issue, who either still maintain the framework, or know who does, but without receiving any substantive response to date.
I resorted to trying to reverse engineer the design and data flow via the published source code.

Initially, this proved difficult; like any good object-oriented program, it relies on a significant amount of polymorphism to function.
I looked through the code, line by line.
Sometimes, I had to guess which package or class to look in next, due to the nature of polymorphism.
I never did find the spot where my tests were invoked;
however, I did locate the place where the framework should have discovered which tests to actually run.
In case you're curious, the problem resided in the code responsible for test class discovery.
Even if I did find the dispatcher, I'd've spent a fair amount time working back to this point anyway.

This story ends happily, thankfully.
But did it have to take this long?
I argue it didn't have to.

## My History

But, first, another historical digression to establish some context.

For decades now, I ardently promulgated and supported the agile cause.
Since the days of extreme programming, I used to believe in the agile manifesto's second credo, "Working software over documentation,"
especially after I let the authors convince me of the value of self-documenting code in the book, Extreme Programming Installed.
Why bother with extensive commentary when the code can comment itself?
Years after writing my own code this way, even without any flavor of comments, I found I was able to still read and understand my work.
Clearly, Kent Beck and friends were right.

I failed to understand, though, that value of self-documenting code applies only as long as you retain any amount of state from working on the project.
Since everything I write for myself is, by definition, *my* state, it follows therefore that I can still grok my code that I wrote way back in 1995.
I feel two reasons explain why this works:
Common coding conventions and a common lexicon of terms and the semantics that goes with them.
A big, big problem happens, however, whenever I attempt to read someone else's code, for which I have no prior project exposure.
It might shine as an exemplar of clean code, even with the occasional helpful comment sprinkled in for good measure;
I'd still find that if it wasn't written in assembly language, I couldn't make sense of it.
All these years, the coding community pushed for and focused on having coding convention standards (PEP-8, go fmt, et. al.), and yes, they're *really* nice to have.
However, I claim, albeit without proof, that it's the lexiconal differences that does the real damage to your productivity.
PEP-8 did nothing to help me find what I was looking for with the testing tool, for instance.
Learning the various lexiconal patterns used in the codebase proved far more valuable.

"Whoa, wait a minute!", I hear you say as you suddenly realize what I'd written.  "What does assembly language have to do with anything?"
Surprisingly, a lot;
however, if you'll indulge me, I'll try to show why later.
For now, just keep it in the back of your mind.

As time passed, I frequently found myself utterly confounded by code written by others.
After so many years of this phenomenon, I started to blame myself whenever this happened.
I'd get really quite depressed about it.
I always feel so inadequate as a software engineer because I can't follow someone else's code.
I mean, everyone *else* can read this code and figure out what it does, so why can't I?

Even when an author followed Beck's advice on writing self-documenting code,
figuring out how one part of a program related to another ranged from extremely difficult to utterly impossible once the scale of the program grew too big.
Too often, I found myself asking, "What purpose does this code serve?  Why do I care about it?  *Should* I care about it?"
This problem only became worse with the popularity boom of object-oriented software, thanks to polymorphism.
(I completely gave up trying to understand aspect-oriented software.)
So it is with the testing tool I've been tasked to use.
I must admit some embarrassment, for during its incubatory year of development, their developers sought my involvement to help shape its evolution.

## My Epiphany

In order for the second credo to hold any value, developers must uphold the first credo just as seriously, if not moreso:
"Individuals and interactions over processes and tools."
Most engineers, without further qualifications, automatically take this to mean "with teammates or the stakeholders."
They almost never consider it in the context of fellow employees in *other* teams; because, you know, they're smart enough to figure it out on their own.
They're engineers, after all!

This is, I feel, the agile movement's Achilles' Heel.
Not only are we not all extroverts, but we just can't always afford to conduct face-to-face interaction.
How many people do you know who've worked hard on a project ABC, seen it through to a successful deployment,
moved onto a different project DEF, and suddenly became unavailable for even the most basic of support queries concerning ABC?
It's happened to me at Cari.Net, at least thrice at Google, at Attributor, twice at Ning, and now here at Rackspace.
Paradoxically, I received the worst treatment at Google, perhaps one of the most pro-agile bunch of coders I've ever worked with:
one of the three engineers I sought help from to understand some old code actually put an auto-responder on his e-mail address and stopped answering his phone after I tried contacting him.
Now; that, right there, is exemplary teamwork.

Having experienced the outcome of my folly, I now consider this behavior, however accidental, just shy of irresponsible not only to your fellow partners but also to all future partners as well.
How many times have you seen months-old (or older) support tickets open which paraphrases as, "Tried to contact engineer about possible bug in ABC, but wasn't available.  Escalating."
Or, tickets reporting a genuine, reproducible bug that has gone unresolved for several *years*.
Here's an actual, real-world example:
how many times have you seen this output produced from Firefox running under Linux,
even though it's now a decade old?

    (firefox:2932): Gtk-CRITICAL **: IA__gtk_clipboard_set_width_data: assertion `targets != NULL` failed
    ** (firefox:2932): CRITICAL **: gst_app_src_set_size: assertion `GST_IS_APP_SRC (appsrc)` failed

Probably never, since most run it from a menu or double-clicking an icon.
Try running it from a console, though, and see what happens.
After a single day's worth of browsing, your console will fill with messages such as these.
Yet, Firefox obviously works; so why bother going through all that low-level implementation detail just to fix an innocuous non-bug like this, right?
(Though, at least these messages include line numbers to help in the debugging effort.)

## My Proposition

OK, I think I've laid out adequate examples demonstrating why I think lack of code documentation is a very real problem.
I know I'm not the only one suffering from it, either.
Looking at the academic and professional literature alike, I see *continuing* work in tricks and techniques to help facilitate rapid code comprehension.
We are even starting to see entire conferences devoted to software documentation efforts.
While many made positive impacts (e.g., generating API documentation via tools like Doxygen or JavaDoc),
few as yet have attempted to tackle the workplace productivity losses due to code comprehension issues.

So, what do we do about it?

I'd like to propose the use of *literate programming*.
Yes, *that* literate programming, the one everyone seems to hate, the one developed and advocated by Donald Knuth.

Remember when I said that assembly listings rarely posed a problem for me?
So many find assembly opaque that assembly coders adopted a very discursive style of commenting.
Most assembly coders demanded high-quality documentation; coders never considered it a luxury.
It wasn't uncommon to look at some random piece of assembly and, from the context of the comments alone, figure out what, how, and why a program did what it did.

Experience shows, however, that commentary-inside-code tends to have more limitations with most contemporary programming languages than code-inside-commentary.
The commentary must appear along with the code in the order that the language demands it.
This makes great sense for API documentation, but not so much for exposition covering how the software works, or why it has the design it does.
Historically, such documentation, if it exists at all, exists out-of-band from the code, which means it's almost always out of date as soon as the next code commit.

Little did I know that better tools existed for since the early 1980s.
While I'd love to go into a full demonstration of available literate programming technologies today,
that lies beyond the scope of this article.
Instead, I want to go into *why* I claim we should adopt to this style of code commentary.

## My Rationale

By far, I find the code-walkthrough aspect of a literate program its greatest value proposition.
Most literate programming tools enforce (or, at least, strongly encourage) a structure to your documents, roughly described by this EBNF:

    Document = Preamble (Explanation Code)*

Of course, some tools allow exceptions now and again; but, all in all, this format seems the most successful structure for literate programs today.
When you think about it, though, it makes perfect sense.
When you stand in front of your fellow team members during a code walk-through,
you often point out a region of code, then talk about it.
People ask questions about it,
and then when it's time to move on, you repeat the process: highlight a new region of code, and start talking about *that*.

Whether you're standing in front of a code review panel, or writing prose in a literate program,
it often helps to start talking about your code with a high-level overview first, and then refine down to lower-level details on an as-needed basis.
Note that this often opposes the bottom-up approach to product development.
Literate Programming enables you to work bottom up, while encouraging a top-down dissertation on how the software works.

Some advantages offered by literate programs over traditional code reviews exist.
For instance, your software can come with a table of contents, allowing people to jump directly to the code that most interests them.
With a traditional code walk-through, people sit through the whole meeting,
while you're wasting the majority's time because only one or two people in the group take interest in any part of your program at a time.
Also on the list of benefits, you're free to include helpful diagrams and other explanatory figures in your explanation.
When I give code walk-throughs, I often have to draw diagrams on the whiteboard, taking the attention of my reviewers away from the code.
Last, but not least, every fragment of code you write sits physically adjacent to its corresponding documentation.
Too many times I've performed code reviews and walk-throughs where I wanted to talk about something, and either didn't have the time, or forgot completely.
Because many tools will flag an undocumented code chunk as an error, the probability that I've forgotten to document some important topic drops asymptotically to zero.

A repository of literate programs largely decouples me from their respective authors.
Organizations should find obvious value from this.
Now, I'm not here to villify who don't want or have time to chat with me about some esoteric aspect of a project long forgotten;
many of them are actually good friends of mine and have quite valid reasons for not responding to me.
One must ask, though, at what point do my priorities influence theirs?
If they'd written the testing tool using a literate style and published the source code "documentation," the probability I'd've been blocked would've dropped precipitously.

It also avoids the game of telephone: every time you recall how the program works, you'll find the details will change.
I can't tell how many times I've wanted to talk about a project at work, and every person I mention it to gets, inadvertently, a different story.
The same thing happens during code reviews.
With a literate program, the story you write in the documentation remains intact, regardless of how many copies are downloaded, printed, or whatever.

## My Conclusion

Look, I'm not asking for Knuth's "book-publishable" or "camera-ready" quality documentation.
While I agree they're really nice to look at, I think people who focus on that aspect of literate programming are missing the point to the wider array of benefits.
I only need enough information to let me do my job.
By facilitating asynchrony between different teams, code documented in a literate programming manner can help encourage a self-serve engineering atmosphere.
It can also significantly reduce new employee ramp-up times, for it makes "read the code" actually meaningful.
This can potentially cut expenses because the time invested writing the documentation in the first place amortizes over every team which depends on it.
Because the documentation sits adjacent the relevant code, the probability of it remaining current exceeds more traditional, out-of-band mechanisms of documentation, such as wiki pages and the like.

I hope I adequately made the case for adopting some flavor of literate programming in professional software engineering organizations.
Thanks for your time.
