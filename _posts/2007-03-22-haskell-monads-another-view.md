---
layout: post
title: "Haskell Monads: Another View (Repost)"
date: 2007-03-22 15:00:00
categories: haskell monads state explanation
---
</p>
 This article was first published 2007 March 22, and is converted from a doctools document.  Some translation errors may remain.  I've tested viewing this page under both Linux and Macintosh environments, with Chrome and Firefox.  (I do not have access to Internet Explorer, sorry.)
</p>
<h1>1  Introduction</h1>
<p>When I first started coding in Haskell, not one month ago, the concept of
<em>monads</em> just thoroughly confused the heck out of me.  Sure, like any clueless
newbie, I could code in Haskell without knowledge of monads.  But who wants to
use a gun without knowing how to clear the chamber first?  Who wants to cook
without knowing how the stove works?  Who wants to <em>learn Haskell</em> without
knowing about monads?  After all -- it's only natural!</p>
<p>After much frustration, I finally came to know them well enough to write this
article.</p>
<h1>2  Monads as Constructors</h1>
<p>To grossly over-simplify, monadic expressions are <em>constructors.</em>  You read
that right -- <em>constructors</em>.  As in object constructors.  To understand why
this is the case, we need to look at the simplest possible monad: the <em>State
Monad.</em></p>
<p>OK, scenario one: you are trying to LR-parse data from some input stream, and
you need to maintain some kind of state while iterating over the input.  Let's
conveniently ignore <tt>unfoldr</tt> for the time being; you're inexperienced, and
you are trying to apply your knowledge of how to code something in C towards
the Haskell solution.  At least, that is, for this example.</p>
<p>Scenario two: you're writing a window manager for the X Window System, and you
need to maintain the list of windows currently visible on the screen.  How do
you do this in a purely function environment, especially when you <em>also</em> need
to process events from a multitude of different sources, parse (<em>ahem</em>)
configuration files, etc?</p>
<p>Due to the genericity of the problem of maintaining state, I won't get into
specific data structures for such state.  Instead, I will concentrate plumbing
required to make the magic of maintaining state happen in a functional
language.</p>
<h1>3  Observation on Sequential Evaluation</h1>
<p>In any functional language, you <em>cannot</em> guarantee the order of execution of
functions.  At least, you're not <em>supposed</em> to.  Just think: according to all
the math texts out there, the results of a function magically appear only once
all its inputs are satisfied.  Functions are supposed to &quot;just work.&quot;  There's
no real way to impose an order of evaluation on them.  Or can you?</p>
<p>A function cannot be evaluated meaningfully without providing all the parameter
values it needs to produce a result.  Thus, it follows that you can
artificially impose a sequential evaluation order by <em>threading</em> the output of
one function into the input of another, like this:</p>
<pre>
&gt; let hello w = w ++ &quot;!&quot;
&gt; in  &quot;hello&quot; : (hello &quot;world&quot;) : []
</pre>
<p>The above code looks like a normal list construction, but remember that you
can't build the list until you have the nodes to stuff into it in the first
place.  Hence, it <em>must</em> evaluate the <tt>hello</tt> function before the list can be
fully evaluated.</p>
<h1>4  Observation on Threading State</h1>
<p>Now that we know how to thread the execution of functions properly, we turn our
attention to the obvious problem of maintaining state from one function to the
next.  Well, this is actually pretty simple -- simply pass, and return, that
state as parameters and/or tuples.  For example:</p>
<pre>
&gt; data Stack a = Stack a (Stack a)
&gt; push :: a -&gt; Stack a -&gt; Stack a
&gt; push a stk = Stack a stk
&gt; pop :: Stack a -&gt; (a, Stack a)
&gt; pop (Stack a stk) = (a, stk)
</pre>
<p>Let's ignore errors for the time being.  What is important to observe in the
above code is that (a) we're passing explicit state (though it's not strictly
true, you can think of <tt>stk</tt> as an object, and its use as analogous to
<tt>self</tt> or <tt>this</tt> in more traditional object oriented languages) at all
times, and (b) the stack &quot;methods&quot; are always <em>returning</em> an updated state as a
result; this is <em>implicit</em> in object oriented languages, where state is updated
in-place.  This isn't necessarily true in functional languages, though
optimizations usually produce equivalent code.</p>
<p>Well, as you can see, having to maintain all that state, manually threading the
output of one function to the input of another, can get substantially tedious!
This can lead to code that is harder to read, even harder to debug, and all but
impossible to modify at some future date.</p>
<p>There has to be some way of factoring this hand-threaded code out.</p>
<p>It turns out that there is.  But first, we need one more observation about the
nature of software before we can put all the pieces together.</p>
<h1>5  Observation on Threading of Flow Control</h1>
<p>In any given imperative programming language, like Python for instance, it's
generally accepted that when you write something like:</p>
<pre>
&gt;&gt;&gt; a = 5
&gt;&gt;&gt; b = 4
&gt;&gt;&gt; c = a+b
&gt;&gt;&gt; print c
</pre>
<p>it is pretty clear that <tt>a</tt> is assigned a value, <em>then</em> <tt>b</tt> is assigned a
value, <em>then</em> the sum of those values are printed to the screen.  In other
words, order of execution is determined <em>exclusively</em> by order of listing.
Control flow follows, strictly, from top down.  Unless told otherwise, of
course, usually by a for loop, or some such.  But those are minor details.</p>
<p>It is pretty clear, in the above code, that we cannot evaluate the value of
<tt>c</tt> meaningfully without <em>first</em> having evaluated both <tt>a</tt> and <tt>b</tt> first.
Sound familiar?  Yes -- that's right -- nested functions.  In this case, if
you'll allow me some literary license, we can rewrite the above code using
lambda expressions in a continuation passing notation, like so:</p>
<pre>
&gt;&gt;&gt; (lambda (a) (lambda (b) (lambda (c) print c)(a+b)))(4)))(5)
</pre>
<p><em>Yikes!</em>  Maybe Python isn't such a good language to express lambda
substitutions with bound variables in.  Indeed, lambdas are going away in
Python in the future, so we might as well look at the equivalent code in a
language that directly supports the notion:</p>
<pre>
&gt; show $ (\a -&gt; (\b -&gt; (\c -&gt; c) (a+b)) 4) 5
</pre>
<p>What, this still isn't clear?  OK, let me show you again, only this time we'll
take it statement by statement.  Or, rather, their equivalents.</p>
<pre>
&gt; show $ (\a -&gt; ...) 5
</pre>
<p>If this looks like normal function calling syntax, that's because it is.  What
we're doing is using a lambda expression to <em>assign</em> the value 5 to the formal
variable <tt>a</tt>.  Pretty slick, eh?  Now, let's look at the next &quot;statement&quot;:</p>
<pre>
&gt; show $ (\a -&gt; (\b -&gt; ...) 4) 5
</pre>
<p>Notice a pattern yet?  Note how &quot;the rest of the program&quot; is treated as a
single function, expressed in terms of the preceding variable assignments?
Note how the only way to successfully evaluate the functions is to evaluate
them in the proper order?  Yes; that's right -- a <em>program,</em> mathematically
speaking, is just a <em>nested set of functions.</em>  Evaluate them in the proper
order, and you will get the same results as an imperative languge program.</p>
<h1>6  Observation: Thinking Algebraically</h1>
<p>By now, you're likely to have had at least one epiphany on where we're going.
If not, I'll now make things more explicit by making one more observation: at
any point in an imperative program, you can always split it <em>between</em>
statements, such that evaluation of the top part is a necessary precondition to
the execution of the bottom part.  It seems pretty obvious at first glance, but
it's a <em>critical</em> observation to make explicit.  Because, after all, if we can
manipulate functions like anything else in higher-order languages, and we can,
then it should be possible to build some <em>function</em> which <em>returns</em> a properly
composed <em>sequence</em> of <em>other</em> functions, that does precisely that.  I mean,
execute A before B, that is.  For the sake of argument, let's call this <tt>&gt;&gt;</tt>.
We can therefore rewrite our Python example like so:</p>
<pre>
(a = 5) &gt;&gt; (b = 4) &gt;&gt; (c = a+b) &gt;&gt; (print c)
</pre>
<p>The associativity of <tt>&gt;&gt;</tt> really doesn't matter; it can be shown that:</p>
<pre>
(a = 5) &gt;&gt; ((b = 4) &gt;&gt; ((c = a+b) &gt;&gt; (print c)))
</pre>
<p>is equal to:</p>
<pre>
(((a = 5) &gt;&gt; (b = 4)) &gt;&gt; (c = a+b)) &gt;&gt; (print c)
</pre>
<p>Hence, we don't usually bother drawing parentheses around such constructs.
Haskell is, by default, left-associative, so Haskell says, also, that &gt;&gt; is
left-associative.  But, strictly speaking, it doesn't have to be.  Anyway,
drawn yet another way:</p>
<pre>
a = 5
    &gt;&gt;      -- Note how these operators are &quot;between&quot; statements
b = 4          conceptually.
    &gt;&gt;
c = a+b
    &gt;&gt;
print c
</pre>
<p>By the definition of <tt>&gt;&gt;</tt>, whatever is on the left-hand (top) side of the
operator <em>must</em> evaluate before the right-hand (bottom) side.</p>
<p>The <tt>&gt;&gt;</tt> operator properly arranges for sequences of code execution, but it
certainly doesn't address that sticky issue of transferring state from one
statement to the next.  How does, after all, one <em>bind</em> variables to values?
Well, remember that dirty trick of nested lambda expressions?  <tt>&gt;&gt;</tt> creates
those, but as shown above, doesn't thread state from one portion of a program
to the next.  Fortunately, we have <tt>&gt;&gt;=</tt> to do that for us:</p>
<pre>
&gt; 5 &gt;&gt;= \a -&gt; 4 &gt;&gt;= \b -&gt; a+b &gt;&gt;= \c -&gt; show c
</pre>
<p>Remember the &quot;we don't care about associativity because it just works out&quot; rule
for <tt>&gt;&gt;</tt>?  It's the same for <tt>&gt;&gt;=</tt> too, since it, too, sequences bits of
the program correctly.  In fact:</p>
<pre>
&gt; a &gt;&gt; b   =  a &gt;&gt;= \_ -&gt; b
</pre>
<p>In other words, <tt>&gt;&gt;</tt> is just a special case of <tt>&gt;&gt;=</tt>, where we don't
particularly care about the result of <tt>a</tt> at all.</p>
<h1>7  Putting it Together: The Not So Obvious</h1>
<p>So, we now have <em>all</em> the pieces we need to finally explain just what the heck
monads are.  So, let's get to work building our very own <em>state</em> monad!</p>
<h2>7.1  Data Types</h2>
<p>We need <em>something</em> to stuff into our state, but we don't know precisely what.
Hence, we're going to use parametric types to describe it.  But one thing <em>is</em>
for sure, we know that properly sequenced functions depends on nested lambda
expressions.  Therefore:</p>
<pre>
&gt; newtype ExState s a = ExS (s -&gt; (s,a))
</pre>
<p>That's right; our data type basically <em>is</em> itself a function.  It takes some
state as input, and returns (hopefully) some value of interest, along with a
modified state.  Remember the observation about how we threaded state as input
and got another state as output?  The plumbing is right there, but is
abstracted in a <tt>newtype</tt>.  Note that a <tt>data</tt> type could work here just as
well.</p>
<p>Next, we need some function to <em>compose</em> pieces of our program together -- we
need to define the <tt>&gt;&gt;=</tt> operator.  It's pretty clear that a program takes
raw input, but provides a transformed version of those inputs in some capacity
(otherwise, what's the point of writing the program?)  Even if the program
provides no useful value for <tt>a</tt>, the fact that it updated some kind of state
<em>somewhere</em> is of great value.  Hence, <em>the result of a program</em> <strong>must</strong> <em>be
of some type of ExState</em>.  Otherwise, what's the point?</p>
<pre>
&gt; (&gt;&gt;=) :: ExState s a -&gt; (a -&gt; ExState s b) -&gt; ExState s b
&gt; top &gt;&gt;= btm           = ExS (\initialState -&gt;
&gt;   let (sTop, vTop)    = perform top initialState
&gt;       (sBtm, vBtm)    = perform (btm vTop) sTop
&gt;       perform (ExS f) = f
&gt;   in  (sBtm, vBtm))
</pre>
<p>Look at what it's doing.  The result of <tt>&gt;&gt;=</tt> is a kind of <em>function</em>, just
like we said it should be before.  It takes an <tt>initialState</tt>, and returns
the pair <tt>(sBtm, vBtm)</tt>, where <tt>vBtm</tt> is the return value from the complete
computation, and <tt>sBtm</tt> is the new state.  And, as both names and intuition
would suggest, these values correspond to the results you'd get by finishing
the program at the <em>bottom</em> of its listing; just like in any other programming
language.  We see that <tt>perform (btm vTop) sTop</tt> is used to compute these
values.  Note that <tt>btm vTop</tt> must, by definition, be some kind of <em>ExState</em>;
we use the helper function <tt>perform</tt> to yank the function out of it.  That
allows us to invoke the bottom part of the program's functionality.</p>
<p>Remember in our earlier discussion how <tt>&gt;&gt;=</tt> bound only a single variable?
Well, it turns out that by doing so, and requiring that function to return a
monadic value itself, we get the following algebraic substitutions:</p>
<pre>
&gt;   let (sBtm, vBtm) = perform (btm vTop) sTop
&gt;   let (sBtm, vBtm) = (\someState -&gt; (anotherState, anotherValue)) sTop
&gt;   let (sBtm, vBtm) = (finalState, finalValue)
</pre>
<p>Pretty slick, eh?  Note the middle line where substitution produces a function.
But where does sTop come from?  Yes; it comes from the first let-binding:</p>
<pre>
&gt;   let (sTop, vTop) = perform top initialState
</pre>
<p>So, in order to properly evaluate the bottom, we <em>must</em> first evaluate the top
portion of the program.  The result of the binding is itself a function which,
upon evaluation, evaluates these &quot;sub-programs&quot; in the proper order.  And, as
you might expect, sub-programs can consist of sub-sub-programs, and
sub-sub-sub-programs, ad infinitum.  The associativity rules of <tt>&gt;&gt;=</tt> allow
us to just keep on threading and threading and threading.</p>
<p>And since we're passing around all these crazy data structures with functions
containing functions containing functions, and states being carefully threaded
from function to function, now you see why I said, earlier in this article,
that monadic expressions are constructors.  What we're building, literally, is
a <em>direct-threaded</em> representation of a program.  The name <em>direct-threaded</em>
isn't accidental; invented in the 1960s and first successfully used by the
Forth programming language, it's purpose was to thread together a bunch of
functions, passing state from function to function.  Just like what we're doing
here!</p>
<p>But, there is one last issue involved with all this: once we are working inside
a monad, how do we actually make use of the state we're maintaining?</p>
<h2>7.2  Accessors</h2>
<p>There are generally two kinds of accessors: <em>getters</em> and <em>setters</em>.  Haskell
makes this patently clear when working with monads, especially State monads,
because the only way to access the &quot;hidden&quot; state is through such accessors.</p>
<pre>
&gt; getState &gt;&gt;= \st -&gt; doSomethingWith st &gt;&gt;= . . .
</pre>
<p>It's clear that <tt>getState</tt> must be a function that returns <tt>ExState</tt> in
this case, since executing it must produce the pair <tt>(sTop, vTop)</tt> inside the
<tt>&gt;&gt;=</tt> function.</p>
<pre>
&gt; getState = ExS (\s -&gt; (s,s))
</pre>
<p>Yes, it really <em>is</em> as simple as that.  What we're doing is we're taking our
state, and returning it as the next return value; we're also leaving it
unchanged in the next state as well.</p>
<p>Changing state is accomplished with a similar, but complimentary, function:</p>
<pre>
&gt; putState x = ExS (\s -&gt; (x,x))
</pre>
<p>Note that we &quot;return&quot; x as well; it's not strictly necessary, since 99.999% of
the time, <tt>putState</tt> would be used with <tt>&gt;&gt;</tt> rather than <tt>&gt;&gt;=</tt>, thus
discarding the result anyway.  But, no matter what, it's patently clear that
the <em>state</em> half of the pair returned is, in fact, being set to <tt>x</tt>;
precisely what we want.</p>
<p>One final &quot;accessor&quot; that really isn't is the <tt>return</tt>.  This is a kind of
hybridized <tt>putState</tt>; its purpose is simple: return a value, leaving the
state otherwise unaltered:</p>
<pre>
&gt; return x = ExS (\s -&gt; (x, s))
</pre>
<h2>7.3  Running &quot;Programs&quot; in the State Monad</h2>
<p>So, let's recap.  We have a set of operators that construct structures in
memory representing what has to be performed (at least conceptually; I should
point out that compilers designed to work with monads often optimize this step
out), at what time, and with what state.  Speaking of state, we have operators
which grant us access to it, for both alteration and query purposes.  We can
construct the cure for world hunger with these basic primitives, but if only we
can actually invoke the programs we create, and get the results!</p>
<p>Once we have the facilities for all that plumbing in place, and we can now
clearly define the concept of &quot;program execution&quot; in terms of simple algebraic
functions, we can now turn our attention to actually <em>doing</em> something useful
with it.  Like, incrementing a counter, for example:</p>
<pre>
&gt; inc = getState &gt;&gt;= \s -&gt; (putState (s+1))
</pre>
<p>Looks pretty straight-forward; admittedly, it's a far cry from curing world
hunger, but it is a start!  In fact, we can thread several &quot;invokations&quot; of
this function along too:</p>
<pre>
&gt; counterIncrementByThree = inc &gt;&gt; inc &gt;&gt; inc
</pre>
<p>Before you know it, we'll be curing <em>AIDS.</em>  But, this syntax, while useful at
times, is pretty ugly and hard to manage overall, so haskell allows us to use
conventional <em>imperative-style</em> programming constructs, called <em>do-notation</em>,
to write clearer code:</p>
<pre>
&gt; inc = do
&gt;         s &lt;- getState
&gt;         putState (s+1)
</pre>
<p>See what's happening?  The <tt>&lt;-</tt> operator maps to the <tt>&gt;&gt;=</tt> operator,
complete with lambda variable binding.  Quite convenient indeed!  But, we
needn't always use variables:</p>
<pre>
&gt; counterIncrementByThree = do
&gt;                             inc
&gt;                             inc
&gt;                             inc
</pre>
<p>In this case, because we're not assigning variables, the Haskell compiler knows
to use the <tt>&gt;&gt;</tt> operator.  But, remember how <tt>&gt;&gt;</tt> was defined in terms of
<tt>&gt;&gt;=</tt> earlier?  That's why we didn't need to <em>explicitly</em> define our own
<tt>&gt;&gt;</tt> operator.</p>
<p>This is all fine, but, the observant reader will point out that we still have
yet to explain how to actually reap the rewards of our programming.  We need to
extract the results of the computation.  This is usually done with a <em>runner</em>.  In fact, we have already seen such a runner:</p>
<pre>
&gt; perform (ExS f) = f
</pre>
<p>That's the one.  Previously, we defined it to be local to the <tt>&gt;&gt;=</tt> operator, and to not bother with the details of passing initial state.  A more complete runner, however, does exactly that -- deal with the state, I mean:</p>
<pre>
&gt; runExS (ExS prg) initialState = snd $ prg initialState
</pre>
<p>Since the result of running a monadically constructed function is a pair
containing (state', result), we use the <tt>snd</tt> function to extract only the
result.  In some cases, you'll be more interested in the state; in this case,
you can define another function if you like:</p>
<pre>
&gt; stateExS (ExS prg) initialState = fst $ prg initialState
</pre>
<p>If both are of interest to you, then instead of evaluating <tt>prg</tt> twice, which
can be a gratuitous waste of time if you're computing the digits of <em>pi</em> to a
billion places, then you can forego the pair selection functions, and just
return the whole pair itself:</p>
<pre>
&gt; valueAndStateExS (ExS prg) initialState = prg initialState
</pre>
<p>Typically, you'd use it something like this in a real-world program:</p>
<pre>
&gt; main = do
&gt;   let (someState, someValue) = valueAndStateExS myCounter 0
&gt;   putStr $ &quot;Resulting value: &quot; ++ (show someValue)
&gt;   putStr $ &quot;Resulting state: &quot; ++ (show someState)
</pre>
<p>Doesn't look like much, does it?  But note that <tt>0</tt> hanging off to the right
on the call to <tt>valueAndStateExS</tt>?  That is the program's <em>initial state</em> --
in C or C++, this is equivalent to the global state, usually established by
numeric constants in main(), or through global (static, hopefully!) variables
in some module.</p>
<h1>8  Conclusion</h1>
<p>Now that you know how the state monad works, you can apply this concept to
any other monad, including the IO monad.  There are other kinds of monads that
are <em>not</em> stateful or state-like though.  These include the list monad (<tt>[]</tt>)
and specialized data types like <tt>Maybe</tt>.  But, for now, I'll let these
specializations of the concept rest.  You've already been through a lot, and
I'm itching to get this paper online.  And, besides, having gone through all
this work, you can now <em>finally</em> appreciate the <tt>Control.Monad.State</tt> and
related libraries.</p>
<p>Well, there you have it; I must bring this essay to a close now, knowing that
you have read yet another tutorial on monads and what they actually are.  My
essay differs from most of the others in that it doesn't invoke the concept of
containers, which apparently has confused a lot of people.  It also doesn't
invoke any complex mathematics (like category theory, where monads come from).
Instead, I resort to normal, day-to-day programming experience, possessed by
any programmer of any imperative programming language.  I hope that this
explanation has proven as useful to you, as it has to me.</p>
<h1>9  Appendix</h1>
<p>The software contained in previous sections are valid Haskell fragments of
code.  However, they don't tell the <em>whole</em> story.  Some details I've had to
leave out for brevity's sake.  But fear not -- contained herein is the complete
program that allowed me to write this essay.  It's pretty short and dense, but
by following along with the earlier parts of this essay, hopefully you'll be
able to see how all the data flows fit together with the monadic plumbing.  I
should also warn you, this code is far from optimal -- it's written so that it
works, and is clear.  If you look at, e.g., <tt>Control.Monad.State</tt> library,
you'll find a substantially terser definition, that does the same basic things.</p>
<pre>
&gt; newtype ExState s a = ExS (s -&gt; (s,a))
&gt;
&gt; instance Monad (ExState s) where
&gt;   top &gt;&gt;= btm = ExS (\initialState -&gt;
&gt;                   let (vTop, sTop)    = perform top initialState
&gt;                       (vBtm, sBtm)    = perform (btm vTop) sTop
&gt;                       perform (ExS f) = f
&gt;                   in  (vBtm, sBtm))
&gt;
&gt;   return x      = ExS (\initialState -&gt; (x, initialState))
&gt;
&gt; getState   = ExS (\initialState -&gt; (initialState, initialState))
&gt; putState x = ExS (\initialState -&gt; (x, x))
&gt;
&gt; stateExS (ExS prg) initialState         = fst $ prg initialState
&gt; runExS (ExS prg) initialState           = snd $ prg initialState
&gt; valueAndStateExS (ExS prg) initialState = prg initialState
</pre>
<h1>10  Acknowledgements</h1>
<p>I would like to take this time to thank the folks in the #Haskell IRC channel
for opening my eyes to this topic.  I could not have come to this understanding
without their input.  I especially would like to single out Cale, Dolio, Kowey,
and SamB, whose patience with me has been the only thing keeping my interest in
understanding monads alive.</p>
<p>Special thanks to Cale and Dolio for reviewing this document.  Their
contributions were invaluable at helping to clarify various points.</p>
<p>And, oddly and finally, thanks to James Burke, for providing a literary style
that I continuously endeavor to match.  It's highly conversational, and very
engaging; it's not the dreary doldrum you'd expect from a paper on, of all
things, stuff called &quot;Monads.&quot;  I mean, c'mon, who ever heard of <em>monads?</em> They
sound like medical conditions!  Anyway, his creative use of <em>suspense</em> and his
masterful <em>touch</em> of humor in educational literature (as distinct from
<em>academic</em> literature) lures the reader into <em>wanting</em> to learn more, which is
precisely the effect I'm looking for.  I hope I've succeeded.</p>
