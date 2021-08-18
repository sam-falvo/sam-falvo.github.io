---
layout: post
title: "On Subroutine Threading for the W65C816 Processor"
author: Samuel A. Falvo II
---

When implementing a subroutine threaded, dual-stack virtual machine for the W65C816 processor,
it might be non-obvious how to attain the highest possible performance.
The roles you assign to registers with the 65816 can significantly impact run-time performance.
Conventional implementations typically prefer a particular register allotment, but without explaining why.
I compare two competing methods of implementing subroutine-threaded code for this processor architecture.
I can finally explain why there exists only a single preferred embodiment of subroutine-threaded code on the 65816 processor.

# Introduction

Running a stack-architecture virtual machine on the W65C816 processor,
such as what you find with the Forth programming language,
incurs significant run-time overhead.
Operations that the processor can natively perform in under 5 clock cycles
consistently consume between 20 and 80 clock cycles, depending upon implementation technique used.
A number of different techniques have been developed to minimize interpreter inefficiencies,
filling out a spectrum of size/speed tradeoffs.
If space consumption is not a primary concern,
one of the fastest methods of running such a virtual machine is to compile stack architecture code
into a representation called *subroutine threaded code*.

A quick review of the 65816 processor's registers that are relevant to subroutine-threading will be helpful,
especially for readers unfamiliar with the 65816 programming model.

    15          0
    +-----------+
    |     A     |
    +-----------+
    |     X     |
    +-----------+
    |     Y     |
    +-----------+
    |     S     |
    +-----------+
    |     PC    |
    +-----------+

Note that other registers which the 65816 supports,
such as the bank registers and direct page register,
are not listed, as they are not relevant for our purposes.

- The accumulator (A) is the sole data processing register.  It can add, subtract, etc.; however, it cannot not address memory.
- The index registers (X, Y) are address offset registers.  Used with a base address of zero, they can be thought of as pure address registers.  However, they can only be incremented and decremented.  They cannot participate in address arithmetic directly.
- The stack pointer (S) is used to maintain the processor's hardware stack.  It works similarly to the index registers, in that it can only be incremented and decremented.  Unlike the index registers, the CPU itself maintains this register automatically.
- Finally, the program counter (PC) always points to the instruction the processor intends to fetch next.

As you can see, the 65816 is severely register-starved compared to contemporary processors.
It is even register-starved compared to an ideal Forth CPU's register set!
Even so, we can still make use of two registers as our virtual machine's stack pointers.
Because the *dp,X* addressing mode only supports the X register,
there are two ways of doing this:

1. The X register can hold the data stack pointer, and the S register the return stack pointer; or,
2. The X register can hold the return stack pointer, and the S register the data stack pointer.

Which of these approaches is better isn't obvious at first.
To help find out, I will take a look at both implementation techniques and
compare their performance relative to each other
by examining a hypothetical compiler's output after consuming Forth code at different levels of abstraction.

# The Problem

Conventional subroutine threading techniques typically allocates the S register to the roll of the return stack pointer,
and X to the roll of the data stack pointer.
This allows the data stack to be overlaid with the direct page segment,
allowing the *dp,X* addressing mode to be used for addressing the stack.
However, the X register must be maintained manually in software,
incrementing and decrementing frequently as the data stack is more frequently used than the return stack.[3]

Consider how a VM implementer might program a simple binary addition primitive:

    LISTING 1.

        jsr enter_add  ; 6 cycles
        ...
    enter_add:
        lda 0,x        ; 5 cycles
        clc            ; 2 cycles
        adc 2,x        ; 5 cycles
        sta 2,x        ; 5 cycles
        inx            ; 2 cycles
        inx            ; 2 cycles
        rts            ; 6 cycles

According the the instruction set listings in [2],
the above fragment of code should take 33 cycles to complete a 16-bit addition, including the JSR needed to invoke it.
Since the addition itself takes 7 cycles (CLC; ADC combination), the rest is overhead.
Can this overhead be reduced if we swap the conventional roles of the X and S registers?

# The Idea

It would be logical to think exploiting the processor's built-in stack hardware
to implement the most frequently used stack
would make the greatest amount of sense.
Pushes and pops perform stores and loads from the stack,
while the processor decrements or increments the stack pointer for us.
Since a typical stack architecture is expected to use the data stack much more frequently than the return stack,
wouldn't it make more sense to let S refer to the data stack instead of the return stack?
The X register, the pointer for the software-managed stack,
would then be used for the least-frequently used resource,
where cost of manual pointer adjustments would be more easily amortized.

# The Details

Performance of 65816 machine code can depend on a number of factors.
To establish a baseline for comparison, I make some assumptions about the runtime environment.
All code discussed herein assumes that we're compiling a 16-bit dialect of Forth,
that the Forth dictionary and its stacks reside in the first 64KiB of memory,
that the direct page base register is aligned to a 256-byte boundary,
that the CPU is in native-mode, and
that all registers are in 16-bit mode.
This allows the processor to run the Forth code in the fewest number of cycles.

Regardless of which register allotment is used,
a null subroutine costs only 12 clock cycles:
the JSR to invoke it, and a single RTS to return from it.

    LISTING 2.

        jsr enter_null
        ....
    enter_null:
        rts

However, this isn't a useful metric, since no work is achieved.
Thus, we must look first to the smallest unit of useful work in the VM: the *primitive*.
Listing 1, which you've seen before, illustrates a simple, 16-bit addition primitive using conventional subroutine threading register assignments.
Recall that it took 33 cycles to run, including subroutine call overhead imposed by the processor itself.

Consider now a subroutine threaded implementation with the rolls of X and S reversed.
Since the `jsr` and `rts` instructions use the built-in hardware stack for accessing return addresses,
and since we're now using that stack for data instead of return addresses,
we need to introduce prolog and epilog code with each subroutine
to ferry return addresses between the data and return stacks.

We start with primitives because they represent the simplest virtual machine unit of functionality.
They have the fore-knowledge that they won't be calling other subroutines,
and so can take some liberties with their implementation to optimize for performance.
Let's re-implement our addition primitive; but, with S and X register roles swapped:

    LISTING 3.

        jsr enter_add  ; 6 cycles
        ...
    enter_add:
        ply            ; 5 cycles
        pla            ; 5 cycles
        clc            ; 2 cycles
        adc 1,s        ; 5 cycles
        sta 1,s        ; 5 cycles
        phy            ; 4 cycles
        rts            ; 6 cycles

In this case, since we don't need it for anything else at the moment,
we use the Y register to cache the return address temporarily while the addition takes place.
This implementation takes 38 clock cycles to complete.
Compared with the code in listing 1 at 33 clock cycles,
this implementation takes a 15% performance hit.

Not all operations in a program consists of strings of primitives, however.
Forth software development practices, for instance, favors highly factored code.
Words can call out to other words, which call other words, and so forth, until primitives are encountered at the leaves of the call tree.
(This all assumes a non-optimizing compiler.)

We cannot just pop the return address into a register and hope for the best,
since a subsequent level of nesting would destroy the cached return address.
This is where the software-managed return stack comes into play.
For example, let's say we want to write a word that adds four numbers together.

    : sum4 ( n1 n2 n3 n4 -- n )   + + + ;

This might be compiled as follows using a conventional subroutine-threaded compiler:

    LISTING 4.

        jsr enter_sum4  ; 6 cycles
        ...
    enter_sum4:
        jsr enter_add   ; 33 cycles
        jsr enter_add   ; 33 cycles
        jsr enter_add   ; 33 cycles
        rts             ; 6 cycles

Total execution time for this fragment of code is 111 clock cycles.

If we swap the roles of X and S,
we find the resulting fragment must take the following form:

    LISTING 5.

        jsr enter_sum4  ; 6 cycles
        ...
    enter_sum4:
        pla             ; 5 cycles
        sta 0,x         ; 5 cycles
        dex             ; 2 cycles
        dex             ; 2 cycles

        jsr enter_add   ; 38 cycles
        jsr enter_add   ; 38 cycles
        jsr enter_add   ; 38 cycles

        inx             ; 2 cycles
        inx             ; 2 cycles
        lda 0,x         ; 5 cycles
        pha             ; 4 cycles
        rts             ; 6 cycles

This comes to a total run-time of 153 cycles, a 37% performance hit.

Interestingly, the performance impact does not appear to compound;
in fact, evidence seems to suggest it will settle somewhere near the 33% mark.
If we go one level of abstraction higher, in an attempt to sum 10 numbers instead of just 4:

    : sum10 ( n1 .. n10 -- n ) sum4 sum4 sum4 ;

we can compare the two register allotment methods.
First, conventional subroutine threading, which comes in at 345 cycles.

    LISTING 6.

        jsr enter_sum10  ; 6 cycles
        ...
    enter_sum10:
        jsr enter_add4   ; 111 cycles
        jsr enter_add4   ; 111 cycles
        jsr enter_add4   ; 111 cycles
        rts              ; 6 cycles

Now, let's example the swapped register allotment approach:

    LISTING 7.

        jsr enter_sum10  ; 6 cycles
        ...
    enter_sum10:
        pla              ; 5 cycles
        sta 0,x          ; 5 cycles
        dex              ; 2 cycles
        dex              ; 2 cycles

        jsr enter_add4   ; 153 cycles
        jsr enter_add4   ; 153 cycles
        jsr enter_add4   ; 153 cycles

        inx              ; 2 cycles
        inx              ; 2 cycles
        lda 0,x          ; 5 cycles
        pha              ; 4 cycles
        rts              ; 6 cycles

This version takes 498 clock cycles to complete, a 30% performance regression start to finish.

You're probably wondering about simply swapping stack pointers and pushing and popping accordingly.
It is an approach that works wonderfully on the Intel 8086 architecture, for example.
However, the 65816 lacks a swap-stack-pointer instruction, so this approach is not cost effective.
The closest we can do is synthesize the effect using a series of register-to-register transfer instructions.
For example:

    LISTING 8.

        jsr enter_sum4  ; 6 cycles
        ...
    enter_sum4:
        ply             ; 5 cycles
        tsa             ; 2 cycles
        txs             ; 2 cycles
        phy             ; 4 cycles
        tsx             ; 2 cycles
        tas             ; 2 cycles

        jsr enter_add   ; 38 cycles
        jsr enter_add   ; 38 cycles
        jsr enter_add   ; 38 cycles

        tsa             ; 2 cycles
        txs             ; 2 cycles
        ply             ; 5 cycles
        tsx             ; 2 cycles
        tas             ; 2 cycles
        phy             ; 4 cycles
        rts             ; 6 cycles

This code would take 160 clock cycles to run, almost 4.6% slower than the code in listing 5.

# Discussion

So, why is the counter-intuitive approach of placing the data stack pointer in X so much faster?
I've identified two reasons which interact with each other.

1. When the S register is used to point into the return stack, prolog and epilog code become virtually unnecessary.  With S and X's roles swapped, the smallest prolog and epilog achievable, that of a `ply` and `phy` pair found in most primitives, contributes 9 cycles to *every* primitive call.  Higher-level words are demonstrated to have steeper costs.  With S pointing into the return stack, however, *all* of these costs are saved.
2. When the X register is used to point into the data stack, the *dp,X* addressing mode may be used to elide unnecessary pointer adjustments until the very end of the subroutine.  For example, notice how in listing 1 we do not adjust the X register until the very end of the primitive.

This is essentially the same phenomenon that can be observed with contemporary RISC architecture processors,
where prolog and epilog code consists of a single adjustment to a stack pointer,
followed by repeated reads or writes into memory relative to the adjusted stack pointer.

    addi sp,sp,-16
    sw   s0,0(sp)
    sw   s1,4(sp)
    sw   s2,8(sp)
    sw   ra,12(sp)

    ; productive computation here

    lw   ra,12(sp)
    lw   s2,8(sp)
    lw   s1,4(sp)
    lw   s0,0(sp)
    addi sp,sp,16
    jalr x0,0(ra)

You'd think that it'd take longer to run such code,
but it in fact doesn't because it has the knock-on effect of freeing the interior of the procedure of all bookkeeping responsibilities.
This amortizes the cost of stack management over the cost of the procedure's more productive computational task.

# Related Work

Subroutine threading isn't the only approach to executing a stack-based virtual machine.
Indirect, direct, and token threaded code representations also exist,
and have been successfully used to implement a variety of different languages for constrained systems. [3]
However, for the 65816 processor, none of these alternative approaches can come close to subroutine threading for performance.
What they lack in performance, however, they make up for in code compactness, which explains their popularity on smaller systems.
You can learn more about these alternative techniques in [3].

# Conclusion

Counter-intuitively,
the conventional approach to implementing subroutine threaded code on the 65816 processor is, indeed, the superior approach.
It works because it consumes fewer cycles to accomplish the same computations
thanks to fewer instructions executed both in between subroutines as well as within them.
Although this study is not exhaustive for all types of software,
it is believed to generalize to the entire virtual machine,
as more sophsticated software is hierarchically composed of these lower-level primitives.

# References

[2] Eyes, David and Ron Lichty.  _Programming the 65816_.  1986.  ISBN 0-89303-789-3.

[3] Rodriguez, Brad.  _Moving Forth_.  Accessed 2021 Aug 17.  http://www.bradrodriguez.com/papers/moving1.htm

