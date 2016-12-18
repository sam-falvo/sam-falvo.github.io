---
layout: post
title: "Some Thoughts on Forth vis-a-vis Oracle and Java SE"
author: "Samuel A. Falvo II"
---

This whole Java SE bullshido that Oracle is all up on their high horse about
has made me think about why I really enjoy Forth.
I'd like to lay down some of my thoughts before I forget them,
because I think it's worth preserving and discussing.

One of the reasons why Forth has suffered in the greater computing community is,
"If you've seen one Forth, you've seen one Forth."
Coming about in an age before the Internet, and with such a huge variety of platforms then available,
standardization proved difficult and, to some, even undesirable.
Yet, in light of Oracle's brutish behavior concerning Java SE,
I find Forth and what little of its ecosystem remains of its former prominence *comforting.*
I'm going to argue that its unbridled freedom of choice is actually its core strength and saving grace.
I'm going to go on record saying that
Forth will outlive Java, even if in relative obscurity to something else.

Forth has been used for everything ranging from stand-up video arcade games to aerospace applications, and
literally and figuratively, everything in between.
[This page](https://www.forth.com/resources/forth-apps/)
lists some if its current terrestrial and extra-terrestrial applications.
Yet, this small but impressive portfolio exists despite the lack of standard libraries or modularity constructs of any kind.
How is this even possible, and why haven't these things evolved over the 46 years Forth's been alive?
Who is actually in *charge* of Forth?  What does it *mean* for something to be called Forth at all?

Honestly, we Forth programmers don't know the answers either.  Here's what I do know.

Forth encourages libertarian thinking about programming,
its own language definition,
and especially its runtime environment.
Forth gestated when you carried your own deck of punch cards for commonly used library routines from mainframe to mainframe,
and of course, every mainframe was different, even those from the same vendor.
IBM 1401, 7904, and System/360 systems all used unique operating systems, filing systems, and calling conventions.
Then there were Univacs, and Honeywells, and Burroughs, and a whole litany of other computers, all special snowflakes of their own.
Consider that each of these systems typically had an average of three competing operating systems at any given time,
it's only natural that one uniquely identifiable tenet of Forth,
"Reduce your dependencies at all costs," evolves.
You might hear of several pithy alternative quotes today:
"Identify and remove non-problems" is perhaps the most recent incantation;
but, you'll also hear about "Write your own subroutines" quite frequently too.
This philosophy enabled easy portability of the language not only between mainframes,
but also between minicomputers, and eventually, 8-bit home computers as well.

With all these different platforms,
you'd expect Forth to be this grand abstraction that hides every detail of a computer from the programmer.
Somewhat counter-intuitively, this is not the case.
Forth also embraces diversity.
Forth implementations seek to rationally expose, not abstract, the unique hardware features of the computer it runs on.
This is not to say that abstractions are universally frowned upon;
Forth implementations are not exokernels.
Yet, as a general rule,
abstractions are treated as adaptation layers that sits between your desired application and the surrounding environment.
Often, these layers are bundled with the *application*, and not provided as a standard feature of the language.

Put more plainly,
*the Forth environment trusts the application to do the right thing, but the application never trusts the Forth environment.*
This differs starkly from the usual view where
the application trusts the runtime or operating system,
but the runtime environment or operating system does not trust the application.

When you have total freedom over the programming landscape like this, won't chaos ensue?
Yes and no.
It's quite clear that without some flavor of standardization,
you cannot take a program for one Forth and run it as-is on another.
Some standards do exist:

1. Forth depends on a two-stack environment: a data stack for evaluation, and a return stack to hold continuations.
2. Forth depends on a dictionary that can grow or shrink, logically similar to a stack, if not in fact.
3. You define new words with **:** (colon), and terminate their definitions with **;** (semicolon).
4. You can define words in such a way that they execute at compile-time ("immediate" words) as well as run-time.
5. You can manually switch from compile-mode to interpret-mode and back again (e.g., with **[** and **]**).
6. Some method of off-line storage is provided, such that it allows you to (at run-time) swap in new code at will.
7. Some method exists of recycling space in the dictionary.

Note the specific absence of *detailed methods* of accomplishing or implementing these attributes.
A typical C programmer will look at this and say, "That's nice, but without more detail, you can't easily port a program."
That's because this programmer is not used to how a Forth environment works.

As someone with first-hand experience porting non-ANSI Forth programs to ANSI Forth and vice versa,
porting software is simply not as big of a problem as it is for, say, a system built around a typical C toolchain.
Forth translates directly from source code to binary with no intervening link editing steps,
and thus concerns of *binary compatibility* generally doesn't arise.
Combined with the fact that most Forth systems make use of a
[hyper-static global environment,](http://wiki.c2.com/?HyperStaticGlobalEnvironment)
it's often more than sufficient to just introduce a *compatibility layer* ahead of the program you actually want to run.
In other words, it's often easy to "virtualize" a figForth environment on an ANSI Forth system,
and it's often as easy to "virtualize" a sufficient subset of an ANS Forth environment on eForth 1.0.
There are, of course, always exceptions.
If your Forth environment lacks the ANSI Forth WORD-LIST facility,
but the application you want to run uses those features extensively,
you'll need to find a way to emulate them.
This takes a lot more work than simply writing wrappers around words with different names or permuted inputs or outputs.
But, honestly, this sort of thing happens only very rarely.
It's usually cheaper to modify the application itself so as to not require that dependency in the first place.
To this day,
a majority of the Forth programming community is able to research and utilize Forth contributions
written long before ANSI Forth ever existed,
going as far back as Forth-79 even,
on contemporary Forth deployments with only minor adjustments.

I invite you to contrast this user experience with that typically found in a POSIX environment.
If you've ever sat next to an ops or site reliability engineer
(both of which I'll collectively lump together as SREs in this article),
you'll see the surface benefits of standardization right away:
people using off-the-shelf software components to accomplish a specific company mandate, and
using generic, off-the-shelf automation tools to bind them together in a "deployment".
Huzzah for software reuse!

What's not made manifest is the months of learning that went into making that happen.
Consider how much an SRE costs a typical company, now consider how many SREs exist in that company.
Few ever consider that the very *need* for an SRE at all is a cost of standardization.
Just look at the tools the SRE is forced to work with,
and read up on their documentation, if any such documentation exists at all.
Look how *much* documentation exists,
and how much of it must be understood before being able to competently use the described tool.
Look on any typical Apache Foundation or Github repository for such tools,
and by reading its front matter, answer me these questions:

1.  What problem does this tool actually solve?
2.  Why is this a problem in the first place?
3.  How does this tool actually solve that problem?
4.  Why should I care?  How does this tool compare with other (known) tools?

If you cannot answer *all four* of these questions from reading the front-matter of a tool's documentation,
I guarantee you'll be spending months to years learning about the tool one way or the other,
and how the tool makes increasingly less sense to continue using.
I've seen this happen time and time and time again in commercial industry.
After all that SRE investment, you're stuck with it.
As the cold, hard reality sets in,
the original engineers responsible for the decision typically bails for greener pastures in a new start-up or project.
That's where you now come in, and realize the gravity of the mess you're stuck with,
and you start to rationalize it as "guaranteed employment."
Tell me this hasn't happened to you even once.

Don't get me started on conferences entirely dedicated to these tools, either.
Seriously: if they're that easy to use,
that learnable,
that reliable,
and provide that much value to their clients,
why do you need conferences to support the community?
Think carefully about that the next time you attend Hadoop Summit or DockerCon.
Pay *close* attention to what's really happening at these conferences.
It's not about you or developing your knowledge; those are secondary effects at best.
It's all about big business:
the vendors, the workshops with said vendors talking about stuff they're working on and/or selling today, etc.
It's not about you at all, and it sure as hell isn't about your customers either.

Major Forth applications are written with the operational demands of its user(s) in mind,
requiring a significantly reduced need for administration overhead.
This allows the developers and the businesses alike to just get on with what they both do best.
Generalized solutions are written for a generalized, platonic, non-existent organization,
and you, the SRE, are left holding the bag in an attempt to get things to work per your organization's business rules.
You're not developing.  You're not creating.  You're duct-taping.

Forth developers generally believe standards bind much more strongly to the communications between modules
(on-disk formats, line encodings, etc.)
than to the modules themselves.
What happens inside the computer *stays* inside the computer, unless there's a documented need to know otherwise.
If you're skeptical that this model doesn't work, you haven't been paying attention.
Consider the Internet itself, built on the compatibility guarantees promised by "RFCs".
This web page you're reading right now
is served from a computer very different from your own,
with a different operating system,
and is connected via infrastructure which doesn't even use the same kinds of microprocessor technology.
If you're the kind of person who believes in the sanctity of IETF standards and how they support the Internet,
then you have a good understanding already of how a typical Forth programmer views the world.
It's not uncommon to find a Forth programmer caring about how a module is implemented
in direct proportion to how broken its interface is.
We don't want APIs.  We want standards.  We'll take APIs if that's all we're given, but we much prefer to make our own.

That gives us freedom:
freedom to create,
freedom to learn and understand the nature of the problem being solved,
and perhaps most importantly,
the freedom to use our software as we see fit without legal interference from some obsolete has-been in the computer industry.
Remember how SCO's final days went down?

