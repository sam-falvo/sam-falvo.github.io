---
layout: post
title: "Some Thoughts on Defined Processes"
author: "Samuel A. Falvo II"
---

# Some Thoughts on Defined Processes

As my role with my current employer evolves, I find it increasingly difficult to remember even the most mundane tasks that I need to perform.  I'm sure [age has something to do with it](https://en.wikipedia.org/wiki/Memory_and_aging), but I strongly suspect that it has more to do with [The Magical Number Seven, Plus or Minus Two.](https://en.wikipedia.org/wiki/The_Magical_Number_Seven,_Plus_or_Minus_Two)  Between remembering reading about the value of checklists in Watts Humphrey's [A Discipline for Software Engineering](http://www.amazon.com/Discipline-Software-Engineering-Watts-Humphrey/dp/0201546108) and the new emphasis on data-driven methods here at work, I decided to follow Watts' advice.  I've tried other methods before, even [taking the time to hand-write my tasks on a daily basis](http://www.medicaldaily.com/why-using-pen-and-paper-not-laptops-boosts-memory-writing-notes-helps-recall-concepts-ability-268770), but nothing so far has delivered *consistent* results.  I'm hoping having a regular checklist will at least allow me to consciously decide when to and when not to perform a task, versus simply *forgetting* all the time.

## It's Not Skill, It's Something Else.

There was a time when I was an ardent agile process supporter.  Why shouldn't've I been?  It revolutionized how I wrote software; I was able to produce clean, relatively bug-free software in relatively short order.  More importantly, it was a *repeatable* process.  I no longer had to *explicitly think* about quality -- it just sort of happened on its own.  I promoted it everywhere I went, and actively did my work using it, even in uncooperative organizations.

Today, I'm finding my ability to keep track of lots of details atrophy slowly yet surely with each passing day.  I'm faced with a harsh reality: getting old sucks, and while I'm still not old enough to be thought of as "old" by society's standards, I'm definitely starting to wake up to the realities of my age bracket.  Keeping track of big-picture architectural details, necessary to know how to navigate the code you're developing, would over time become quite fuzzy.  I often find myself spending one or two *days* just trying to remember how to do something important to my task at hand.

Ironically, while my sensitivity to details seems to be reducing with age, my *skill* level doesn't appear to change all that much overall.  Once I have a firm understanding of what I need to produce or perform, I can execute with the same alacrity I had when I was 20 years younger.  I still understand the basic principles of what I do every day.  So the question is, how do I compensate not for faltering skill, but rather for my faltering ability to process large amounts of information seemingly at the same time?

I'm forced to re-evaluate a topic which I've not only discarded in the past, but actively campaigned against in my distant youth: *disciplined processes.*  Of course, I still campaign against the top-down mandate for such processes; however, if implemented bottom-up, then by definition such processes are fit-for-purpose.  They can then deliver (and have delivered) many benefits to teams and individuals alike.

While I'm obviously inspired by Watts Humphrey's [Personal Software Process](https://en.wikipedia.org/wiki/Personal_software_process), which even Humphrey himself admitted is a high discipline process, please be aware that this is *me* taking the plunge, not my employer.  *I'm* doing this because *I* feel inadequately capable of keeping up with my peers and delivering *with confidence* value to my fellow team members.

## It Starts With a Checklist.

Yesterday, I wrote my first personal process checklist.  It's a script which I follow every Monday.  It actually contains a list of rather mundane things, including but not limited to the following:

* Look for specific e-mails and copy their contents to other, well-known wiki pages.  This is something that ought to be easy to automate, but experience suggests that manually spending the time once a week is actually cheaper than investing the time to automate it.
* Look at all Github issues opened since the previous week, and assign severity and priority labels to them.  This has to more to do with the company's recent push for data-driven metrics on defect rates than with any particular value that it delivers to the team; hence, why we assign these asynchronously from issue creation.
* Look at my own Github issue history, and using that information, compose our own version of the infamous [TPS report](https://en.wikipedia.org/wiki/TPS_report), the equally infamous [PPP report](https://en.wikipedia.org/wiki/Progress,_plans,_problems).  In case you're wondering, PPP stands for Progress, Plans, Problems; exactly the kinds of things you discuss during a daily stand-up meeting, but then for some reason have to repeat once more in a more formal manner as you report back to your manager what you did that previous week.
* Schedule semi-regular meetings with the engineering managers, and re-assess current priorities so as to make sure that the QA department can deliver maximum value to the product department at all times.
* Check for vacation time, and if I have any coming up, make sure I register it with the company.

and so it goes.  I will, if need be, devise scripts for other days of the week as well.

Why bother, though?  Wouldn't a calendar application serve just as well to offer reminders?  If this works for you, then great.  For me, however, the answer is a solid no; the events I plug into a calendar are transient at best, even when configured to be recurring.  There's no such thing as "reading my calendar for the day"; I quickly forget what's scheduled after I read it.  (While writing that sentence, in fact, I had to check my calendar to make sure I wasn't about to miss a meeting.  This would have been the fourth time I checked it today.)  The reminders are only helpful if I can see them.  *If I'm away from my computer, then I'll have no conscious memory of any pending events.*  Anyone can create invites without my knowledge or permission.  Especially if management creates an invite, I cannot often just say "no, I'm busy".  Usually, all I can do is reschedule.  Finally, calendars are really poor vehicles to communicate exactly *how* to perform a certain task.  You really want a channel that properly supports rich, long-form communicaitons.

So, for me, calendars are transient, out of sight/out of mind, often incontrovertible, short-form, and asynchronous in exactly the wrong sort of way.  They're useful; don't get me wrong!  I don't think I could function as efficiently without one.  Yet, they clearly aren't covering all the use-cases for helping me get my day-to-day activities done.  So far, I'm finding checklists to be more helpful for those kinds of rote, daily tasks.

## Context Switching

I find checklists are also excellent at supporting context switching.  Let's suppose I need to go to a meeting (from one of those calendar invites I was talking about).  After I come back from the meeting, which might last for an hour, I then have to pick up where I left off.  What do I do now?  With a suitable checklist, I may not be able to completely switch contexts as ideally as I'd like, but it'll be a whole heck of a lot faster than if I didn't have it.  My Mondays are interrupted by several meetings, so getting my daily tasks done required several such context swaps.  My checklist already proved its worth.

## Estimating Size, Resources.

This is more an issue for non-rote tasks.  I'm not estimating any task sizes or resources yet; but, it's something I'd like to start doing for all my non-rote, deliverable-oriented tasks.  From these estimates, I can then build simplistic [Earned Value charts](https://en.wikipedia.org/wiki/Earned_value_management) which I can then use to help inform my management of when things will be complete.

I actually have some experience using EVM and task breakdowns to predict delivery dates on two previous occasions.  First, I used it to plan out my own work in developing some aspects of the Kestrel-2.  Second, I used it to develop a functioning prototype "Project Eagle Eye", a mechanism that combined my team's documentation with programming examples and our CI/CD infrastructure to always test the example codes in a CI/CD manner.  My goal was to help ensure our technical examples were always correct.  While both efforts demonstrated excellent fidelity to the initially planned completion dates, I'm particularly proud of Eagle-Eye, where I delivered the proof of concept within 2 days of the planned delivery date.  That was a schedule slip of only 1.11%.

I tried doing this with a sub-project for the Kestrel-3 recently, only to discover half-way through the project that I'd done the EVM wrong.  Whoops.  By sheer luck, it turns out that I was able to continue to schedule things, but the report it produces communicates the wrong metrics.  (The only reason it still works is because my basis for comparison used the same units.)  Had I used a checklist, the probability that I'd make this mistake would have been minimized, or even null.

A necessary prerequisite for getting EVM to work for you is to [break tasks down](https://en.wikipedia.org/wiki/Work_breakdown_structure) into atomic (or nearly atomic) units of work, and estimate how long they'll take to complete (PSP recommends using task hours, but you could just as well use gummy bears or other abstract units as long as they're used *consistently*).  The smaller the unit of work, the better.  This is something I'm notoriously bad at; so much so, I'm surprised I'm still employed at all.  I typically end up decomposing a task into units of work which are not atomic in the slightest, and that usually means I take a lot longer than expected to complete a high-level task.  After being in the industry for so long, you'd think I'd've learned by now.

The fact that I *haven't* learned this, however, seems to suggest that this activity is *also* best addressed with a checklist of some kind.  What are the common criteria that are used to decompose tasks?  Record them in a checklist, and make sure to reference the checklist as often as possible, so that I can remember to look for those facets.  Obviously, if the checklist fails me, then during a retrospective, revise the checklist for next time.

## Conclusion

Clearly, using a defined process has worked for me on several occasions in limited capacities, so it's baffling to me why I bothered going back to working in a more ad-hoc manner instead of increasing my adoption of the recommended practices.  It's clear that I need to get back into the habit.  It's like dieting, really; once you're on a diet, you're *always* on that diet.  As soon as you diverge even a little bit, you regain the weight you worked so hard to shed.  So it is with personal processes: the moment you diverge from a proven personal software development process, the wheels start to fall off the wagon.  Whether dieting or adhering to a defined process, the real challenge can be found in sticking with it.
I'm hoping I can stick with it this time.

