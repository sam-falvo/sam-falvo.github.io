---
layout: post
title: The Declarative, Imperative, then Inquisitive Pattern
author: Samuel A. Falvo II
date: 2010-02-27 16:30:00
publish: true
---

<div class="alert alert-info">
 <p>
  Based on <a href="http://www.forth.org/svfig/kk/02-2010-Falvo.pdf" class="alert-link">a presentation</a> I gave at the <a href="http://www.forth.org/svfig" class="alert-link">Silicon Valley Forth Interest Group</a>,
  I wrote this article some time before 2010 Feb 27; however, I don't recall exactly when.
  I originally published it on one of my earlier blogs, now long since gone.
  The content, including all errors, remains intact from its original publication; I made no attempt to clean up the prose below.
  I only edited it so that it may conform to contemporary formatting methods (e.g., Markdown).
  While details have changed over the years, the core essence of this pattern remains as true today as it did when I first published it.
  Maybe some day, I'll revisit this pattern and post an updated pattern reference, complete with examples in a variety of languages instead of focusing exclusively on Forth.
  Until then, enjoy this piece of personal archaeology.
  More will be coming as time permits.
 </p>
 <p class="pull-right">
  &mdash; <em>Samuel A. Falvo II, 2013-Dec-07.</em>
 </p>
 <p>&nbsp;</p> <!-- needed to overcome weirdness with pull-right class.  Admittedly, I *am* abusing it in the context of a paragraph, but still... -->
</div>

Many interacting tensions need resolution if one desires a "well-written" Forth program.
Unfortunately, except for the relatively scattered and often contradictory tips and suggestions offered in the book *Thinking Forth*,
documentation of common problems and their solutions hardly exists.
To remedy this problem, I present a programming pattern which I call **Declarative, Imperative, then Inquisitive.**
Hopefully, this article will inspire others to contribute their own patterns.

<!-- more -->

# Name

Declarative, Imperative, then Inquisitive (DItI).


# Problem

Reasoning about software is extremely difficult.
Any code can potentially cause any change to the state of the machine;
conversely, any change the code makes might (or might not!) possibly be intentional, even if it doesn't match the function's name.
Therefore, every programmer learns the importance of clearly naming their procedures, and many coders learn to use code without side effects.
But, the side effects manifested in one procedure might be the intended main effect of another, and much benefit often exists in allowing side effects.

We therefore define a pattern which clearly marks procedures as being either Declarative, Imperative, or Inquisitive.
A Declarative procedure states (in its name) that a given action or state is accomplished or reached;
its semantics should ensure that the named action is accomplished or state is achieved,
and furthermore that the action will only be accomplished at most once (that is, if the procedure is called multiple times with the same state, no further action will be taken).
An Imperative procedure commands (by its name) that an action be done, and its semantics should perform that action every time it's called, regardless of whether it was already done.
An Inquisitive procedure states a question, and it should do nothing more than return the answer to that question, not changing the state of the system.

Furthermore, we specify that whenever possible, actions should be specified in the most limited manner possible, which means that most words should be Declarative rather than Imperative;
and because Inquisitive words do nothing but allow decisions to be made, they should appear relatively rarely in order to reduce the complexity of the system.
Thus, we list the names in their desired order of frequency.

Allow me to demonstrate with a more concrete example.

When I first started the HDLC networking component of my digital optical transceiver project,
I needed a data store for the local/remote station connection relationships.
Implementing this required a simple means of locating a record based on remote and local addresses (which I dubbed `row`, for it conceptually returned a database row).
But, I did not want to deal with error conditions at that point &mdash;
this meant that my queries *must always* produce the intended value, no matter what.
I would handle errors elsewhere, where it proved more convenient.
That kept my program logic clean and readable, unfettered by irrelevant logic.

When I first conceived `row`, I didn't think in terms of declarative coding techniques;
rather, I thought of the query word as a *procedure*;
e.g., something I dictated to the computer: first do this, then do that, finally do those.
The procedural thinking I first used yielded the following definition (`dropDlc` is the procedure that calls `row` in this case, hence the bizarre comment on the 7th line):

    : row ( ra.la )
      0 >r
      begin ( ra.la : 0 <= rel <= nextDlc )
          r@ nextDlc @ = if
              2drop r> drop
              0 r> drop
              ( exit dropDlc w/ f : 0 <= rel = nextDlc )
              exit
          then
          ( ra.la : 0 <= rel < nextDlc )
          over r@ remoteA + @ =
          over r@ localA + @ = and if
              ( ra.la : [0 <= rel < nextDlc] /\ isDlc?[ra.la] )
              2drop r>
              ( rel : [0 <= rel < nextDlc] )
              exit
          then
          ( ra.la : 0 <= rel < nextDlc ==> 0 < rel+/row <= nextDlc )
          r> cell+ >r
          ( ra.la : 0 <  rel <= nextDlc )
          ( ra.la : 0 <= rel <= nextDlc )
      again ; 

While I was quite pleased with how easy it was to use `row`,
I was not at all happy with how complex this word turned out to write.
`row` seemed irreducable to me at the time<sup>1</sup>,
for I could find no meaningful way to simplify it.
To help ensure the word's correctness,
I placed proof annotations interstitially in the body of the definition.
The word definitely works &mdash; I've proved it "on paper",
and it also worked spectacularly well in practice.

Later, I encountered a most difficult bug where incoming connections were not appropriately accounted for in the data link connection state (DLCS) table;
after hours of trying to track the bug down, process of elimination indicated that the bug had to reside in one of two places:
inside `row`, or somewhere inside the frame dispatcher module.
Despite the formal proof that this code worked,
I decided it was the most likely cause of the failure,
due to its visual complexity (maybe I had forgoten something).
Just in case, I decided to rewrite it.<sup>2</sup>

This time, however, I wanted to re-implement `row` using definitions of similar structure to those using `row` itself,
for I had discovered that using `row` in other definitions proved remarkably easy,
and made for very readable code.
I started with the most obvious use-case:
I knew that if a row wasn't found in the DLCS table, we had to return a zero to the caller of whatever word invoked `row`
(remember: `row` cannot itself return without having a valid record number as its result).

    : row           -found  2drop 0 r> drop ;

A word about notation: since ASCII lacks the boolean symbol for logical negation (this thing: &not;),
I'm forced to choose the character with the closest iconographic resemblence:
words starting with a dash usually read as, "*not* word" or "*no* word."
In this case, `-found` reads as *not found*.

Notice the structure of the aforementioned definition.
`-found`, taken otherwise completely out of context, *states a fact or pre-condition* about the code which follows,
which the reader safely assumes must hold for all subsequent code.
Hence, `2drop 0 r> drop` executes with *full confidence* that the record sought is genuinely not in the database.

Eventually, I had to implement `-found`.
Once again, I decided to engineer the code so that it stated only facts,
without obvious concern to what would happen had these facts been wrong:

    : -found        0 begin dup nextDlc @ < while -match cell+ repeat drop ;

For those not familiar with idiomatic Forth coding conventions, I defined `-found` to mean, literally, *for all allocated records in the table, no match exists.*
Note that `-found`, while itself a declaration, also makes use of another declaration: `-match`.
`cell+` runs with full confidence that no match has been discovered thus far.

    : -match        hit? if nip nip r> r> 2drop exit then ;

`-match` ensures that no match ("hit") exists insofar as it concerns `-found` and `row`.
What happens, though, if it discovers a match?
Considering the context we've established so far, we *cannot* just return to the caller because the caller's subsequent code *depends* on there not being any match!
Likewise, we cannot return to the word who called `row`'s caller.
Only returning to the word *which called `row` itself* remains, making sure we also return the record number we promised for subsequent code to use.

It was at this time I realized how widely applicable declarative programming in Forth can be.
From examples like `2 4 connected` to establish the relationship that remote station 2 and local station 4 were connected to each other,
to using `row` to guarantee a database record number,
to using preconditions in a declarative mode, as above,
I now had a consistent pattern of declarative programming wherein the legibility of code significantly improved while also improving code reliability at the same time.

# Context

Declaration, Imperative, then Inquisitive applies whenever you desire:

 * easy to read and maintain source code.  The best Forth code tends to read horizontally, not vertically, through the use of the rule of thumb, "One line, one definition."  DItI provides a more structured means of achieving this goal.
 * greater code reliability.  Tony Hoare was one of the first computer scientists to identify the concept of *preconditions*, and later popularized by the Eiffel programming language through its *Design By Contract* system.  He defined a precondition as a predicate which must hold true for any subsequent software to produce valid, correct results.  Declaratively documenting preconditions at the beginning of Forth words both documents the requirement and provides a means to trap on erroneous input.

# Forces

 * Readability &mdash; The DItI pattern, in effect, calls for a coding convention.  Inasmuch, as with all conventions, familiarity with it results in measurable improvements in reading and comprehending unfamiliar pieces of code.
 * Correctness &mdash; The DItI pattern improves correctness by encouraging input parameter checking through preconditions.
 * Performance &mdash; In naive compilers, a subroutine call will occur for every declaration, even for those used only once.  Hence, depending on your compiler, using DItI may impact runtime performance in timing sensitive event handlers or tight loops.

# Solution

We have already observed the structure of `row`.  I repeat the code fragment below, sans interstitial comments for greater clarity:


    : hit?          >r over remoteA r@ + @ =  over localA r@ + @ = and r> swap ;
    : -match        hit? if nip nip r> r> 2drop exit then ;
    : -found        0 begin dup nextDlc @ < while -match cell+ repeat drop ;
    : row           -found  2drop 0 r> drop ; 

Notice how `row`, `-found`, and `-match` exist as declarations &mdash; these words state or establish some truth, which code that uses them can rely on.  However, `hit?` is inquisitive in nature.

Note the following conventions:


<table class="table table-bordered table-responsive">
 <tr>
  <th>Word&nbsp;Type</th>
  <th>Attributes</th>
 </tr>
 <tr>
  <td>Inquisitive</td>
  <td>
   <ul>
    <li>Typically uses a question mark to ask a question.</li>
    <li>Typically past- or present-tense.  E.g., <code>connect<b>ed</b>?</code>, <code>reus<b>able</b>?</code>.</li>
    <li><b>Always</b> idempotent.  Assuming no other externally induced state changes (including but not limited to time-sensitive properties, including the current time itself), invoking a predicate with the same parameters <b>must</b> return the same results.</li>
   </ul>
  </td>
 </tr>
 <tr>
  <td>Imperative</td>
  <td>
   <ul>
    <li>Typically named with a command-phrase.  E.g., <code>sortArray</code>, <code>printError</code>.</li>
    <li>Per the principle of command/query separation, imperatives almost never return anything to their callers.</li>
    <li>Rarely idempotent.  E.g., when printing, <code>newPage newPage</code> should cause the current page to finish, followed by a blank page.</li>
   </ul>
  </td>
 </tr>
 <tr>
  <td>Declarative</td>
  <td>
   <ul>
    <li>Typically named as a verb-derived adjective.  With the most common form of expression as a past participle form of a verb (ending in <b>-ed</b>, as in <code>connect<b>ed</b></code>), we understand the program state to reflect the results of some previous action and remains so to the present time.  Depending on the context, you may find a present progressive form more suitable (ending in <b>-ing</b>, as in <code>connect<b>ing</b></code>).  Still other forms, perhaps more rarely encountered depending on the kind of software written, suitable names appear as a verb with some other suffix that makes it adjectival (such as <code>reus<b>able</b></code>)<sup>3</sup>.</li>
    <li>Most declarations have eponymously named queries.</li>
    <li><b>Always</b> idempotent.  Although declarative words may effect new state, re-asserting the same truth more than once has no further effect.  E.g., <code>1 2 connected</code> will establish the fact that 1 and 2 are somehow connected.  However, re-executing the expression will, in effect, do nothing, for the computer already knows that 1 and 2 are connected.</li>
    <li>A fact exists in at least one of two possible times: before a declaration executes, or after it's finished executing.  As a result, declarations come in two basic forms: preconditions, which <b>confirms</b> facts known ahead of time, or state changing, which performs requisite actions to <b>effect</b> new knowledge.</li>
    <li>Preconditions typically <b>do not</b> consume their stack arguments, instead preserving them for subsequent computations in the event that the precondition holds.  If a precondition fails, however, it takes immediate action to handle the exceptional condition, consuming parameters if necessary.</li>
    <li>State changing words <b>do</b> consume all of their stack-resident arguments, often treating the stacked data as a representation to interpret, rather than raw data to store verbatim.  The word takes all actions necessary, <b>if any at all</b>, to alter the relevant state according to the representation given on the data stack.  (C.f., Representational State Transfer, or REST.)</li>
   </ul>
  </td>
 </tr>
</table>

# Resulting Context

* Declarative words ("declarations")
 * always express eponymously-named facts.
 * are *effective* &mdash; internal data and/or control-flow state may change to ensure the named facts actually *are*.
 * are *idempotent* &mdash; they take no unnecessary actions beyond that required to effect their stated truth.
 
* Imperative words ("imperatives")
 * provide the know-how responsible for making queries and declarations work.
 * generally appear at the lower abstraction levels, and therefore remain hidden from external programs.
 * typically deal with the realities of memory layout, pointer arithmetic, etc.
 
* Inquisitive words ("queries")
 * provide a read-only view on the relevant state.
 * can answer yes/no questions (e.g., are there enough bytes to read?) or reconstruct a representation of some internal state (e.g., where is the current cursor position in the Cartesian coordinate system?).  Many declarations have eponymously named queries.
 
* Runtime Performance
 * Potentially compromised due to increased subroutine call overhead.
 * Recovery possible through creative use of immediate and/or macro definitions.
 
* Program Architecture
 * State maintenance most likely requires some form of database.  Any suitable database architecture will work, including but not limited to key-value, object-oriented, relational, hierarchical, navigational, et. al.  Any persistence model will work as well, including disk-backed, distributed RAM cache, or even ordinary record/object member fields.
 * Languages lacking aggregate data types like records or objects tend to rely more heavily on relational(-like) database concepts.  Languages with native support for more sophisticated aggregate types tend to find more navigational styles of data management easier.
 
The programmer takes responsibility for choosing appropriate names for his or her words.
Try to choose names that make sense in the context of the problem being solved, not for the underlying data structures used.

For example, if you have a queue of objects to process, `enqueued` and `enqueued?` likely will make for poor names even though they're academically correct.
Since the enqueue function takes both a datum and a queue to put it on, one must wonder what queue `enqueued` refers to.
Does `enqueued?` query the same queue that `enqueue` uses?

Queues appear in many different kinds of software; even within any single project, any number of queues may exist, each serving a unique purpose.
Therefore, the programmer must recognize this and ask *why* something needs queueing in the first place.
Put another way, if something deposits an object on a queue, what significance lies behind it?
What *becomes true* about the object once it's been queued?
If you experience difficulty answering any one of these questions, remain aware that answers to them always exist;
when found, the answer provides a valuable source of ideas for choosing among candidate names.
Having a thesaurus nearby helps too.

External software tends to use declarative words, usually overwhelmingly due to their ease of use.
Try to minimize reliance on queries, for they exhibit a tendency to break a module's encapsulation.
The most frequently used predicates tend to reflect the safety of performing an action or ability to affect state.

Software tends to read more conversationally, at least once you're used to the adopted notational conventions; it communicates more naturally with the human maintainer, as humans think declaratively.
Contrast against imperative-only coding (both procedural and object-oriented variants), where the communication emphasis lies with the machine, or functional coding, where the emphasis lies with algebraic formulation and evaluation.

# Examples

To help illustrate this pattern, we consider the relatively simple task of tracking a text input cursor on the screen.  Consider a 640x480 pixel display, with an 8-pixel fixed-width, 8-pixel tall font.  This yields a character matrix 80 columns wide, 60 rows tall on the screen.  As a first cut, we know we need to keep track of the cursor's coordinates:

    variable cx
    variable cy
    : at            cy !  cx ! ;
    : at?           cx @ cy @ ;

`at` repositions the cursor on the screen, while `at?` queries its current location.  Because queries may idempotently provide different views on some state, we might want to return a byte offset into a bitmap corresponding to the current cursor location.  We'll define a word `tile` to return the base address of a character tile.  Although a query, notice that `tile` lacks a question mark:

    : tile          cy @ 80 * cx @ + ;

However, the behaviors of `at?` and `tile` makes sense only for the case where (0 &le; *x* &lt; 80) &and; (0 &le; *y* &lt; 60).  If this condition doesn't hold, then we get strange effects, including the possibility of memory corruption elsewhere in the system.  So, let's constrain our coordinate space:

<pre>
: <b>constrained</b>   0 max 59 min  swap  0 max 79 min  swap ;
: at            <b>constrained</b>   cy !  cx ! ;
</pre>

`constrained` demonstrates a declarative word which ensures our precondition by constraining the cursor to the visible bounds of the screen. Notice it takes no unnecessary actions, it leaves the stack as-is for subsequent code, and also deals with exceptional cases aggressively.  It's also unlikely that this word find use by outside software, so choosing a more generic name for it doesn't make sense here.  However, if it's desired to use `constrained` elsewhere, then rename the word to something more appropriate (e.g., `visiblyConstrained`) as a refactoring step.

When editing text, few things occur more frequently than rendering a character and advancing the cursor.  Thus, we can define a word which bumps the cursor to the right:

    : bumped   1 cx +! ;

Of course, this only works when 0 &le; *x* &lt; 79; when *x* = 79, we need to wrap the cursor to the left-side of the screen:

<pre>
: <b>r-edge?</b>   cx @ 79 = ;
: <b>-wrap</b>     <b>r-edge?</b> if 0 cx !  1 cy +!  r> drop then ;
: bumped    <b>-wrap</b>   1 cx +! ;
</pre>

While an improvement, we still neglect the case where *y* = 59.  To prevent our invariant from being violated, we need to refine our code once more:

<pre>
: <b>b-edge?</b>   cy @ 59 = ;
: <b>-scroll</b>   <b>b-edge?</b> if scrolledUp r> r> 2drop then ;
: r-edge?   cx @ 79 = ;
: -wrap     r-edge? if 0 cx !  <b>-scroll</b> 1 cy +!  r> drop then ;
: bumped    -wrap   1 cx +! ;
</pre>

Observe how you can take *any single line of code* and understand it in complete isolation of the others, provided you have the over-arching context of the problem being solved (in this case, bumping the cursor to the right).  You'll find this occurs quite frequently with DItI, and helps to explain why DItI-style code proves easier to maintain.

# Known Uses

Chuck Moore uses some declarative programming throughout ColorForth, albeit with highly abbreviated names often hiding their declarative characteristics.
See [http://www.colorforth.com/ide.html](http://www.colorforth.com/ide.html) for the ColorForth IDE harddrive source code, published circa 2001.
The `bsy` and `rdy` words, responsible for ensuring the IDE controller is not busy and is ready to receive or send data respectively,
fulfill the declarative requirements established in this pattern.
However, `sector`, `read`, and `write` take imperative forms.

Samuel A. Falvo II uses declarative programming style extensively in his HDLC network implementation.

# Related Patterns

At least with Forth, Partial Continuation often appears as the only way to satisfy the declarative coding style.
Relying on sentinel return values, even if in a separate stack item, often complicates program flow.
`CATCH` and `THROW`, while useful in their own right, still work with sentinel values at some point.

I suggest applying Aggressive Handling as well, for by definition, all exceptional cases to documented truths needs dealing with in the word establishing that truth.
Eliminating run-time error dispatching will, in most cases, significantly simplify software maintainability.


# Acknowledgements

I would like to thank Billy Tanksley for offering his extremely limited time to proof-reading this article, and offering valuable input concerning the expression of concepts and ideas herein.

--------

<sup>1</sup> If you strip away the comments of the above definition of `row`, you will find the core logic simple enough.
You'd be forgiven if you, too, thought that refactoring the word would yield no tangible benefits.


<sup>2</sup> It turns out that `row` was *not* in error; the proofs were correct.
However, had I not mistrusted myself, I don't think this pattern would ever have been recognized and documented.


<sup>3</sup> Some might feel that such forms do not make explicit the idempotency of a procedure.
While I do not feel this to be the case (e.g., once something is reus<b>able</b>, it will always remain reusable until such time as it is reus<b>ed</b>), that one person can think this implies others can too.
Regrettably, I cannot prescribe formulaic rules for this; only experience can inform how and when to use such names.

