---
layout: post
title: "Subroutine Performance in J 701b"
author: Samuel A. Falvo II
---

I couldn't find micro-benchmarks on J's subroutine calling performance.
I provide some measurements to fill this gap.
I focus on J's ability to invoke verbs in the context of writing a game,
where I expect to manipulate lots of short vectors in a control-heavy environment.
I also show how it compares to an equivalent program written in GForth.

### Introduction

During Christmas and New Year's vacation, I decided to re-install the J programming language and get back into it.
To help me really learn the language, I'm planning on porting Equilibrium, a game I once originally wrote in Forth.
I'm looking to tackle this challenge more for fun and self-education than to produce anything serious.
This task will require my familiarity with several aspects of J, including but not limited to how to invoke foreign functions,
but also how to write software such that tacit and explicit definitions work together.
This naturally raises the question, how much code should exist tacitly versus explicitly?
Put a slightly different way, how much time does J consume when invoking tacit versus explicit verbs?

While Forth's subroutine calling overhead is well-known and documented for several different platforms,
the same doesn't appear to be the case for J.
I've seen numerous, but unsubstantiated, claims to the high performance of the J interpreter.
I fully recognize subroutine call overhead may not necessarily influence overall program performance;
however, it certainly has an impact on how one writes software in that language.
I remain unaware of any published micro-benchmarks relating to the J programming language at all,
much less one that focuses on subroutine calls.

Considering J's strong emphasis on mathematics,
any published benchmarks will likely involve lengthy vectors or wide matrices, neither of which will exist in Equilibrium.
For instance, the classic demonstration of finding the average of a long sequence of numbers appears in virtually every J tutorial on-line:

    (+/%#) i.100

Indeed, if we widen our vector and time it, we find J apparently runs very fast indeed:

       (6!:2)'(+/%#) i.1e6'
    0.003648

If we amortize the time across the million elements provided by `i.`, we find the computer spent about 3.648 _nanoseconds_ processing each number in the vector.
That includes memory overhead for creating the vector, taking the summation, etc.
This seems impossibly fast, at 274e6 or so numbers processed every second.
My PC runs at 3.4GHz, meaning my machine spent somewhere in the vicinity of 12 cycles for each number.
Remember, this amortization includes memory allocation overhead and other run-time bookkeeping that J performs for you.

Since I write highly factored code and anticipate vectors no longer than 4 atoms in length,
I predict the overhead of invoking functions on short vectors will dominate J's performance, and will slow down appreciably.
This experiment aims to falsify or verify my prediction.

### Problem Statement

As I write this article,
I'm in the process of cloning Spike Dislike, a game available for several mobile platforms by the author Jayenkai, for my Forth-based Kestrel home-made computer.
An important requirement of the game involves moving the spikes across the screen.
With a very simple CPU architecture and lack of sprite hardware,
I need to break down a spike's X-coordinate into two pieces to help make drawing faster:
a 16-pixel column number, and a pixel offset from the left-edge of that column.
This won't necessarily apply for games implemented in J for Linux,
where I intend to rely on the SDL and byte-per-pixel graphics layouts.
Nonetheless, I retain the logic here, since
it's representative of a real-world design decision which directly influences performance on the slower Kestrel architecture.
Equilibrium involves similar logic for its combatants and particle effects.

In summary, we can describe the locations of our four spikes using a vector of coordinate triads:

    0 0 0   --  spike 0
    0 0 0   --  spike 1
    0 0 0   --  spike 2
    0 0 0   --  spike 3
                ...
    | | |
    | | +-----  bitmap row at which it appears
    | +-------  pixel offset (0 <= offset < 16)
    +---------  word column (0 <= column < 40)


With the Kestrel version of Spike Dislike, we have no greater than four spikes in the Kestrel's 640x200, monochrome playfield at any given time.
Thus, we create this matrix simply in J using the shape operation:

    locs =: 4 3$0

Spikes will need to move to the left during game-play.
To facilitate this, an operation called `nudge` exists, which adjusts the X-axis coordinates appropriately.
It's simplest definition, omitting any hypothetical drawing-related code for brevity, appears in listing 1.
We apply it to all spikes using the rank adverb:

    locs =: nudge"1 locs

With Spike Dislike, the only five things that move on the screen include the ball and the four spikes.
However, with Equilibrium, we include particle effects as well, which increases the number of on-screen objects into the hundreds.
The more efficiently J can support processing these individual, 3-element vectors in a loop,
the more objects I can keep on the screen before flicker or sluggishness becomes noticeable to the player.
Determining the best way to accomplish this task, therefore, directly influences player experience.

### Experiment Setup

I used two platforms:
the first is my desktop computer, running under Linux on a four-core Intel 64-bit processor, at 3.4GHz, with 8GB of RAM.
See Listing 4 for one of the four `/proc/cpuinfo` reports corresponding to this computer.
The second platform is a Samsung Galaxy Tab 3, model GT-P5210, equipped with 1GB of memory.

For the PC version of J, I downloaded the version found at [http://www.jsoftware.com/download/j701a_linux64.sh](http://www.jsoftware.com/download/j701a_linux64.sh), accessed December, 2013.
For the Galaxy Tab 3, I run the Android version of J, which I found at [https://github.com/mdykman/jconsole_for_android/blob/master/dist/j-console.apk](https://github.com/mdykman/jconsole_for_android/blob/master/dist/j-console.apk), accessed January, 2014.

To ensure a sufficiently large data set to measure performance of invoking a function on a small vector,
I use a significantly taller matrix of locations than discussed in the introduction.
To ensure the computer keeps busy enough to average out random noise,
I elected empirically to use 1.3 million location tuples, arranged in a cube:

    locs =: 130 10000 3 $ 0

Originally, I used a shape of `1300000 3$0`, but this yielded slightly slower performance.
I'm at a loss to explain why, but it might serve as a topic for future research.

I first started out with the most obvious implementation, that of the explicit script in J.
Structurally, this script looks as you'd expect from any functional programming language.
When moving a spike to the left one pixel, we check for underflow of the pixel offset, and if so, adjust the word column accordingly.

Next, in listing 2, I rewrote the explicit script in tacit form.
`pfn` stands for "point-free nudge."
Each of the conditional cases appear respectively in `pfna` (pixel offset equal to zero) and `pfnb` (pixel offset non-zero).
`mez` tests to see if the pixel offset is zero ("middle equals zero").

At first, I conducted the tests only with these two versions.
However, I later thought about a third case: an explicit definition whose body consists of a tacit implementation.
Listing 3 shows how I arranged the code.
I generated this code by copying the result of `pfn f.` at the JConsole into the new definition, `pfns` (point-free nudge, scripted).

Finally, I created the following mnemonic verb to return the time taken per row in the matrix.

    time=:(6!:2)%1300000"_

With this, I invoked th following code and recorded the resulting figures for each of the platforms considered:

    time'locs=:nudge"1 locs'
    time'locs=:pfn"1 locs'
    time'locs=:(pfn f.)"1 locs'
    time'locs=:pfns"1 locs'

As a point of comparison, I wrote the same software in GForth (see Listing 5).
Due in part to the substantially faster word execution performance over J,
and in part due to the substantially reduced timer precision provided by Unix command-line tools,
I needed to increase the number of rows that `pfn` needs to process to 1.3e9.
This caused GForth to run for just about a full minute on the PC, allowing `time gforth pfn.fs` to report meaningful results.
You'll notice I allocate only 1.3e8 cells of memory in the code, but iterate over the data set ten times.
I tried allocating 1.3e9 cells (5.2GB) directly, but at least with GForth 0.7.0 64-bit running under Linux,
I cannot allocate that much memory.

### Data

Table 1 presents the raw data collected on the PC and Tab architectures.
Table 2 presents the same data normalized to the fastest for each platform.
The data for table 2 was derived from the following J expressions or the PC and the Tab, respectively:

    (]%(<./)) 5.15617e_6 2.56521e_6 1.41128e_6 1.26321e_5
    (]%(<./)) 5.87663e_5 2.63402e_5 1.46162e_5 1.41435e_4

The fastest possible execution happens, at roughly 1.4us per coordinate row on the PC, when you invoke an explicitly flattened, tacit definition.
The next fastest comes from invoking the tacit definition alone, costing almost 1.8 times as much execution time.
In the latter case, I suspect vocabulary look-up for `pfna`, `pfnb`, and `mez` verbs happens for each row.
Flattening performs this look-up once, creating an anonymous verb with the same definition as the named verb, with all dependencies filled in ahead of time.
Thus, by the time the interpreter recognizes all the inputs for the anonymous verb, it only needs to look through a single tree to evaluate it;
this amortizes the cost of looking up `pfn`'s dependencies across all the available rows to process.

The explicit definition `nudge` incurs a 3.6 to 4.0 factor performance penalty, coming in at the 3rd slowest to execute.
J seems to parse and interpret a script, line by line, token by token if not character by character, using an RR parser every time it's invoked.

That a tacit definition inside an explicit definition takes almost an order of magnitude longer to run than a flattened tacit definition truly caught me by surprise.
I originally believed that the lack of complex control flow or the need for vocabulary look-ups would have made parsing and execution performance fall between explicit and tacit without flattening.
Even knowing what I know now, I would still expect the performance hit to not exceed 7.0.
J seems to expend a _significant_ amount of effort when invoking scripts with tacit functions in them.
I speculate this excess work comes from the dynamic synthesis of anonymous verbs and the memory management overhead that entails.

I measured the GForth equivalent software dispatch speed at close to 38ns per row.
It would appear that J requires close to 37 times longer to invoke a verb than Forth does.
My gut tells me this excess latency comes mostly from dynamic memory management overhead.
Forth, as with C and other static languages, operates in-place on memory.
J, however, may need to construct new vectors.
While I'm aware that J recognizes and optimizes in-place updates for the `m}` operation,
it's clear that J misses the same opportunity for in-place updates with a statement such as `locs =: pfn"1 locs`.
My transcript for measuring GForth's performance appears in listing 6.

### Discussion

It should be noted that J _always_ executes what it can, when it can.
J will _immediately_ interpret adverbial phrases, like `{.@}.`, to dynamically construct anonymous verbs at parse-time.
Thus, by the time the interpreter reaches the `=:` copula, the _value_ of the right-hand side will be a complete anonymous verb,
itself consisting of invokations of anonymous sub-verbs, etc.
The leaves of this data flow tree have no dependencies of their own, and so take their inputs from the left- or right-hand side of the expression this anonymous verb finds itself in.
If insufficient inputs exist, the phrase remains in verb form for later processing when all inputs become available.

If bound to a label, such as the case with `pfn`, J doesn't have to reparse and rebuild the tree.
Every reference to `pfn` will place in J's parse stack a reference to the definition that already exists.
However, if executed inline, such as what happens inside a script based on observational evidence,
J would have to resynthesize the anonymous verb every time.

With J's best case taking 37 times longer than Forth's case on the PC hardware,
it's clear that J's claim to high performance must come from exploiting relatively lengthy vectors or wide matrices.
Even an unscientific test seems to confirm this: if we let `xs=:1.3e6$0` and execute `time'<:"0 xs'`, we see an average invokation of 2.9ns per cell.
Wrapping <: into a verb of its own fails to produce a measurable difference in performance.
With data so arranged, the tables turn, with J outperforming Forth by almost an order of magnitude.

### Error Sources

It's possible my understanding of how J works under the hood fails to match reality.
I have studied the J interpreter implementation, but am far from mastering an understanding of it.
Therefore, you should take my speculation of what influences runtime performance with some skepticism.

Factors influencing my performance measurements may not influence your own.
Always make sure to _measure_ your application's performance against your needs.

### Conclusion

I've measured the relative performance of invoking an explicit and tacit verb, each with variations.
Flattened, tacit definitions run the fastest, with each invokation of `pfn` taking 1.4us (est.) on the PC, and 14us (est.) on the Tab 3.
Meanwhile,
tacit definitions expressed inside of explicit definitions run the slowest by close to an order of magnitude.

Based on the findings,
I recommend writing software using tacit definitions where at all possible.
You gain significant reduction of source code size while claiming a performance reward for free.
Try to minimize the use of explicit definitions for processing vectors of data, as they incur an estimated 2x performance penalty.
Instead, reserve fairly simple explicit definitions for control-oriented processing, such as any side-effecting feature of a program.
If you cannot avoid invoking a script over a vector, avoid the use of tacit definitions inside the script at all costs.
Instead, define them tacitly as verbs outside the script, and refer to them by name.
Keep names in scripts short yet mnemonic, since J will parse those names every single time the verb runs.

If performance remains an issue, and assuming most of your software exists in tacit form,
you may employ the `f.` operator to, in essence, "compile" the verb's call-graph into a single abstract tree,
thus roughly doubling run-time performance due to removal of vocabulary look-ups.
This costs a little extra memory, however, as the anonymous verb `f.` creates must duplicate not only the named verb's implementation,
but also the implementation of all its dependencies.
It may also exhibit deleterious effects when attempting to invoke polymorphic methods on a vector of objects,
as the flattened abstract tree `f.` creates may not agree with all object types in the (presumably boxed) list.

Finally, consider re-arranging the layout or representation of your data structures.
As indicated above, J works best with _wide_ vectors or matrices.
For example, in the case of Equilibrium,
my best interest lies with avoiding N&times;3 matrices, and opting instead for 3&times;N matrices.
Better still, use _parallel vectors_ &mdash; arrays synchronized against each other, and explicitly named.
This forms a relational table of sorts, with each variable naming a column of atoms.
Based on my informal tests, such an organization would have improved performance by almost _three_ orders of magnitude in the average case.
Such a reorganization, however, comes at the cost of code legibility, for related data items no longer co-reside in the source code.

### Table 1.  Time taken to process a single row of the `locs` matrix, in seconds.

<table class="table table-bordered table-responsive">
 <tbody><tr>
  <th>&nbsp;</th>
  <th><tt>nudge"1</tt></th>
  <th><tt>pfn"1</tt></th>
  <th><tt>(pfn f.)"1</tt></th>
  <th><tt>pfns"1</tt></th>
 </tr>
 <tr>
  <th>PC</th>
  <td>5.15617e_6</td>
  <td>2.56521e_6</td>
  <td>1.41128e_6</td>
  <td>1.26321e_5</td>
 </tr>
 <tr>
  <th>Galaxy Tab 3</th>
  <td>5.87663e_5</td>
  <td>2.63402e_5</td>
  <td>1.46162e_5</td>
  <td>1.41435e_4</td>
 </tr>
</tbody></table>

### Table 2.  Relative performance versus the different ways to process `locs`.

<table class="table table-bordered table-responsive">
 <tbody><tr>
  <th>&nbsp;</th>
  <th><tt>nudge"1</tt></th>
  <th><tt>pfn"1</tt></th>
  <th><tt>(pfn f.)"1</tt></th>
  <th><tt>pfns"1</tt></th>
 </tr>
 <tr>
  <th>PC</th>
  <td>3.65354</td>
  <td>1.81765</td>
  <td>1.00000</td>
  <td>8.95081</td>
 </tr>
 <tr>
  <th>Galaxy Tab 3</th>
  <td>4.02063</td>
  <td>1.80212</td>
  <td>1.00000</td>
  <td>9.67659</td>
 </tr>
</tbody></table>

### Listing 1.

      nudge =: 3 : 0
    'xc xp yp' =. y
    if. xp=0 do. (xc-1),15,yp
    else. xc,(xp-1),yp end.
    )

### Listing 2.

      mez =: 0&=@({.@}.)
      pfna =: _1&+@{.,15"_,{:
      pfnb =: {.,_1&+@({.@}.),{:
      pfn =: pfnb`pfna@.mez

### Listing 3.

      pfns =: 3 : '({.,_1&+@({.@}.),{:)`(_1&+@{.,15"_,{:)@.(0&=@({.@}.)) y'

      NB. I created this fully-expanded definition by issuing pfn f. at J console,
      NB. and copying result into the script.

### Listing 4.

    processor       : 0
    vendor_id       : GenuineIntel
    cpu family      : 6
    model           : 58
    model name      : Intel(R) Core(TM) i5-3570K CPU @ 3.40GHz
    stepping        : 9
    cpu MHz         : 3401.000
    cache size      : 6144 KB
    physical id     : 0
    siblings        : 4
    core id         : 0
    cpu cores       : 4
    apicid          : 0
    initial apicid  : 0
    fpu             : yes
    fpu_exception   : yes
    cpuid level     : 13
    wp              : yes
    flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx est tm2 ssse3 cx16 xtpr pdcm sse4_1 sse4_2 popcnt aes xsave avx f16c rdrand lahf_lm ida arat epb xsaveopt pln pts dtherm tpr_shadow vnmi flexpriority ept vpid fsgsbase smep erms
    bogomips        : 6820.10
    clflush size    : 64
    cache_alignment : 64
    address sizes   : 36 bits physical, 48 bits virtual
    power management:

### Listing 5.

    3 cells constant /row
    130000000 /row * constant /locs
    /locs allocate throw constant locs
    locs /locs + constant locs)

    : pfna      -1 swap cell+ +! ;
    : pfnb      -1 over +!  15 swap cell+ ! ;
    : mez       cell+ @ 0= ;
    : pfn       dup mez if pfnb exit then pfna ;
    : row+      /row + ;
    : pfn"1     begin dup locs) xor while dup pfn row+ repeat drop ;

    locs pfn"1
    locs pfn"1
    locs pfn"1
    locs pfn"1
    locs pfn"1
    locs pfn"1
    locs pfn"1
    locs pfn"1
    locs pfn"1
    locs pfn"1
    bye

### Listing 6.

    kc5tja@deneb ~ $ time gforth pfn.fs 

    real    0m49.110s
    user    0m40.919s
    sys 0m0.980s
    kc5tja@deneb ~ $ time gforth pfn.fs 

    real    0m48.459s
    user    0m41.027s
    sys 0m0.880s
    kc5tja@deneb ~ $ time gforth pfn.fs 

    real    0m48.907s
    user    0m40.871s
    sys 0m1.024s
    kc5tja@deneb ~ $ time gforth pfn.fs 

    real    0m49.629s
    user    0m40.931s
    sys 0m0.968s
    kc5tja@deneb ~ $ time gforth pfn.fs 

    real    0m50.148s
    user    0m40.999s
    sys 0m0.908s
    kc5tja@deneb ~ $ ./j64-701/bin/jconsole
       (+/%#) 49.110 48.459 48.907 49.629 50.148
    49.2506
       49.2506%1.3e9
    3.78851e_8

