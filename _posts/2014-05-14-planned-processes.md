---
layout: post
title:  "Planned Processes: Not So Evil After All?"
date:   2014-05-14 11:24:37 PDT 2014
categories: PSP TSP planned agile process processes
---

This post is not politically correct in the contemporary software engineering community.
I probably just blogged myself out of any future employment.
But, when has that ever stopped me from saying what needs to be said?

As some of my readers know,
I work at a sizable company called Rackspace.
We're not super-huge; nothing like Google or Microsoft or even IBM.
However, we're not exactly small potatos either.
Historically a company focusing purely on excellent customer support,
Rackspace today pushes itself to become more of a technology company with their OpenStack-related offerings.
Rackspace adopted a (more or less) agile approach to software engineering and product testing alike.
For the last two years, this approach worked wonders.
Rackspace helped define a new market,
and in the face of the recent Oracle-vs-Google ruling,
we and our customers stand to benefit from OpenStack's open-source API and implementation alike.
Life's good!
Or is it?

Last week, my supervisor passed along a mandate that product testing must happen according to
a documented test strategy that conforms to a standard, comprehensive template.
When I received exposure to this template for the first time, I thought it looked quite onerous.
This template includes all manner of compulsory, and occasionally optional, fields to fill in.
While filling in the blanks is simple enough if you have the data at-hand,
it's not always so simple to research the data you need to fill those blanks in with.
Tools exist to help the prepared engineer with this, such as Watts Humphery's Personal Software Process,
or burn-down charts if you're a SCRUM master, so I won't consider this topic further,
except to say that most engineers are not so prepared.
With this new template,
part of the responsibility for project planning now falls squarely on the product's quality engineers (QEs) assigned to that project,
including forecasting which different phases of testing are needed,
when they start and for how long,
what is known to be in- and out-of-scope,
an architecture diagram or description,
and more.
No part of this form seemed to comply with our previously pro-agile approach to product testing.
My gut felt sick, with thoughts of pitchforks and torches dancing in my head.
In short, I panicked.

Now, let's fast-forward to today.
I needed to perform perhaps one of the simplest possible tasks one could do for any project:
contribute changes to my group's customer-facing, online documentation.
It's maintained using Sphinx, and the repository is on Github, so it ought to be a simple task, right?

Never underestimate the power of poor communication.

After spending about an hour and a half researching and repairing a botched Sphinx installation<sup>1</sup>,
I tried for an hour more to get the damned thing to build the documentation website.
Nothing I tried worked, so I issued several pleas to my more senior coworkers for help.
Surprisingly, I was deferred by two senior engineers to speak to another, as they couldn't remember the steps taken to let them edit the docs.
After close to an hour of this, I finally get a helpful response from the third engineer.
It turns out I need to install two repositories, one on top the other, for the build to work correctly.
Tell me again why this isn't written somewhere?

And that's when it hit me like a ton of bricks.
It occurred to me at this moment exactly *why* my organization (which focuses on quality, security, and repeatability across all of Rackspace),
as distinct from my group (which focuses on delivering a single product to our customers),
now requires QEs to fill out strategies using the recommended template.
That form collects the wisdom and hard lessons learned over several years of other quality engineers running into exactly the kinds of problems I did above.
How do I...?  What does ... do?  All I did was .... and the whole system broke!  Why?  Who do I contact if ... ?
And, last but not least, why isn't it written somewhere?

The template embodies a simple philosophy: *pay it forward*.
It recognizes certain repeatable communications failures in teams regardless of their process, be it SCRUM, Lean, or cowboy coding.
Namely, engineers and business-folk tend to ask the same kinds of questions, but engineers are loathe to respond.
By filling in this template for your product, it frees you from having to respond at all in most cases, at least beyond pointing to the document.
In a way, the template is a generalized FAQ.
You just provide the details unique for your product or service.

Some questions are asked so frequently that any time you start a new project,
or if you aim to maintain an existing one,
you just *know* you're going to need to answer them at some point anyway.
Not only that, but you can even predict who is going to ask them, and depending on the demographic, when.
Business folks, those handling the money behind the project's funding, want to know you're making regular and controlled progress.
They want to know that what they're paying for the software (consider engineer salaries, insurance, and benefits) is worth it to the business.
Customer support folks want to know where they can find high-level overviews of the product
so they can answer customer questions in a timely manner.
System administrators want to know who to escalate level-2 or level-3 questions to.
Engineers want to know the nitty-gritty details so that when a bug comes in, they know where to look
without having to waste hours to days narrowing down the subsystems.

Knowing that these questions are coming, you might as well just take the time to answer all of them up-front.
That's exactly what the new template aims to encapsulate.
Spending a few days to a week to perform a good (enough) job at planning can save a lot more time down the road.
If nothing else, treat it like a FAQ.
When junior engineers ask a question that's already answered on the form, just point to the form.
When management wants to know your progress, point them to the form and give a small earned value update, a percentage, or something.
You don't have to wait or dread this.
I know you're busy.  Hell, we're all busy.
But, if a junior engineer asks a question that is already answered by the strategy document, just point to the document then and there.
You'll alleviate the mental burden to answer the question later, and you'll save a crap-ton of the junior engineer's time too.
As if that weren't enough, as engineers come and go from your project, and they will,
you'll ultimately save them countless hours of ramp-up investment.
Brook's Law will still apply, just not as much.
Heck, interested engineers can even read about your product and how it works even before their start date.
How's that for hitting the ground running?
Isn't it worth it to spend a couple of days drawing up the plan?

But what if the plan isn't correct?
Isn't planning up front just another form of big-design-up-front?
News flash!
Plans will **always** be incorrect.
By definition, they're glorified estimates!
As long as (1) it's close enough, and (2) deviations are reported to stakeholders at the time of the discovery, nobody will care if it's technically wrong.
It's been my experience that senior management doesn't generally have a problem with issues that come up in a project.
They only want them to be reported as soon as they're discovered, so that they have enough "runway" to take corrective action.
It turns out that these values, core to many agile processes, also apply to planned processes too.

So what does this have to do with planned processes?
Well, everything.
See, the template is itself a planned process script, by definition.
This could well be Rackspace's first planned process script for software engineers.
The template instructs the reader how to fill it out, even going so far as to provide examples.
Once filled out, the resulting strategy serves as a kind of checklist to the QE, a calendar to business folk, an introduction to the system for new engineers, etc. all at once.
From the examples cited in the template, it does quite an adequate job at it too.
I've learned a lot about several of our internal projects which I had no knowledge of before.
Already, as an engineer, I benefited from the form.

This means, I think, that Rackspace, the technology company, is maturing.
Our products are both stable and mission-critical enough to warrant more rigid processes,
so that our sales and executives can offer commitments to high-stakes customers with agreed-upon service level agreements.
And, unlike Microsoft's products, if our products break, we end up costing our customers *lots* of money, which we may be liable for under some SLAs.
There's no such thing as "just reboot or reinstall the cloud," no matter what anyone tries to tell you.
Obviously, we don't want to put either customers or ourselves in such a costly position.
So, while I doubt we even register on the CMMI maturity scale yet,
that we're organically recognizing the need for greater maturity in our processes can only be recognized as a *good thing.*

Of course, as with anything else, you can always have too much process imposed from on-high.
When management imposes reporting requirements, it becomes a slippery slope.
Let's hope this doesn't become a habit.
Too much process can easily be as stifling as no process at all.
But, for the time being, I look forward to stepping up my game and growing my skills as a professional software engineer.

<hr />
<sup>1</sup>&nbsp; Not Sphinx's or Python's fault; it looks like damage incurred when I upgraded my Mac to Mavericks.  Still, why did it happen in the first place?!  Clearly a quality engineering issue!

