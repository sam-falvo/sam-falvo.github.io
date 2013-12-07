---
layout: post
title: "How Does MISC Stand Up to RISC, CISC? (3/7)"
author: Samuel A. Falvo II
publish: true
---

I have yet to see objective comparisons between MISC, RISC, and CISC architectures.
MISC processors often get a bad rap for their stack-based micro-architectures; however, generally-available evidence to support these negative views seems lacking.
I try to put these processor architectures into perspective using real-world experience with some processors I've used directly in the past.
I hope the data revealed will help others in answering questions on which architecture proves right for them.

In this article, I highlight the F18A core, which comprises the computing elements of the Green Arrays' GA4 and GA144 chips.
Details of this processor may be found [at the Green Arrays website](http://www.greenarraychips.com/home/documents/).

<!-- more -->

### Space Consumption

As implemented in any Green Arrays product, the F18A is incapable of supporting a program as long as what I've coded in this article.
However, there's nothing fundamental about the F18A core architecture that prevents its use with systems containing larger quantities of memory.
The analysis that follows assumes such a hypothetical machine.

As before, the different routines in the program have been sized.  The table below refers to the program found in Listing 1.

```
Name       Size (%)
====================
plt         14  13.3
pltc         5   4.8
zerop (0p)  12  11.4
zeroq (0q)  24  22.9
addrs        2   1.9
flip         5   4.8
crsr        39  37.1
plotch       4   3.8
====================
TOTAL       105 words (236.25 bytes)
====================
```

The F18A demonstrates significantly smaller code than the S16X4, despite an 18-bit vs 16-bit word size discrepency.

I counted 41 uses of the `li` instruction, which means 41 of the 105 words (39%) contains literal information, much of it repeated.
Unlike the S16X4, the F18A supports very rapid subroutine calls and returns in hardware.
Idiomatic development of F18A software would factor out frequently repeated sequences of instructions, such as effective address calculations, into subroutines.
This helps improve code density while incurring only a modest performance penalty.
Listing 2 illustrates how one might refactor listing 1.
The table below illustrates how this refactoring affects the final program size.

```
Name       Size (%)
====================
plt         14  18.2
pltc         5   6.5
_outsfc__    3   3.9
_outsfc_os_x 2   2.6
_outsfc_os_y 2   2.6
zerop (0p)   7   9.1
zeroq (0q)  15  19.5
addrs        2   2.6
flip         3   3.9
crsr        20  26.0
plotch       4   5.2
====================
TOTAL       77 words (173.25 bytes)
====================
```

That simple change yields a 26.7% reduction in program size.

### Runtime Performance

Without exception, *all* F18A instructions takes one instruction cycle to execute.
(Observe that the F18A lacks external clocking, being a fully asynchronous architecture.
Nonetheless, it offers a uniform instruction execution cycle length.)
However, to conserve the maximum amount of energy, the arithmetic unit inside the core relies on ripple-carry; thus, calculating a sum frequently requires more than one cycle.
This explains why you need a `nop` in front of all `add` instructions (unless you can statically prove ripple delays are insignificant).

Unlike the S16X4, the F18A requires that the programmer preload a memory address to fetch from or store to into a special register.
Thus, all memory references require no fewer than two instructions as well.

Finally, as with the S16X4, the CPU takes an additional instruction cycle to fetch the next batch of up to four instructions.
However, with a truncated fourth slot, only a subset of instructions may appear there, often resulting in a word containing no greater than three instructions.
This results in an increased instruction fetch rate.

The following chart refers to the program in listing 1, for it represents the fastest implementation.

```
Name       Cycles (%)
====================
plt          33+(0*0)
pltc         11+(8*(7+plt))
zerop (0p)   29+(0*0)
zeroq (0q)  107+(0*0)
addrs         4+(0*0)+0p+0q
flip         13+(0*0)
crsr         57+(0*0) (worst case)
plotch        8+(0*0)+addrs+crsr+pltc
====================
TOTAL       262+(320) = 582 cycles
====================
```

This table tells us that initializing a bitmap pointer and printing a single character takes 582 cycles to complete, minimum.
Note that I made no attempt to cache structure field references, and yet, the F18A still proves faster than the S16X4 by 4.6%.
I believe the existence of the return stack and its supporting instructions account for the performance boost.
In particular, since the return stack holds return addresses (hence its name), we no longer need to inflict a 7-cycle cost to invoking subroutines to manage return addresses.
Moreover, when using finite loops, the F18A hardware provides the `next` and `unext` instructions, which combine a decrement-counter (which exists at the top of the return stack) and branch-if-non-zero operation into a single instruction.
With the S16X4, I had to manually decrement and branch if non-zero, further adding to a subroutine's latency.

Structure field caching would significantly reduce the fixed overhead of calling `plotch`, from 262 cycles to 191 (a 16.3% improvement over S16X4 baseline), and would actually bring program size down as well.
What's more, it would enable faster character plots after the bitmap calculations have already been done --- additional characters would require only 320 cycles.
Replacing the multiplication in zeroq with a table lookup would save 58 more cycles, reducing the fixed overhead further to 133 cycles (a final boost of 25.7% over baseline).

Finally, a fair amount of memory accesses involve read-modify-write operations (e.g., incrementing `p` and `q`).
With the S16X4, I had to push the addresses onto the data stack each time a memory access became necessary.
With the F18A, the A register caches the effective address, allowing me to amortize a read and write into three instruction cycles (loading A, fetch, then store) versus four on the S16X4 (load address onto stack, fetch, load address again, store).

### Cost of Structure Field Access

Accessing fields off of a base pointer takes eleven cycles and five words to complete:

1.  Fetch the `li base; lda; fwa" bundle.
2.  Spend three clock cycles executing these instructions.
3.  Fetch the `li field_offset; nop; add` bundle.
4.  Spend three cycles executing these instructions.
5.  Fetch the `lda; fwa` bundle.
6.  Spend two cycles executing these instructions.

Accessing an absolute memory location takes three cycles on average, four worst-case, while requiring significantly fewer words of memory to encode.
If the effective address to fetch from or store to remains unchanged, a cycle may be avoided, as the A register already contains the desired address.
Clearly, executing references to absolute memory locations proves much faster, and as well, consumes far less program memory.
Though not illustrated in this example, the F18A also supports accessing memory and post-incrementing the A register.
For some classes of software, this can yield significant savings in time.

If left as-is, the F18A MISC would actually be slower than a Motorola 68000 when reading or writing structure fields.
In our specific example, the savings from having a hardware return stack slightly exceeds the cost of repeated structure field references.

### Cost of Indirect Subroutine Call

Idiomatically, the F18A environment stores return addresses on the hardware return stack.
Implementing an indirect subroutine to any arbitrary address involves pushing a fake return address on the stack, then returning to that address.

```
li      subroutine_pointer
lda
fwa
push
rfs                             ; or 'ex' if you're calling from inside a larger subroutine.
```

This takes seven cycles to complete; since the processor maintains the linkage information on the return stack, no additional time need be spent.

In this case, the S16X4 runs closer to RISC speeds than the F18A does, which still runs faster than classic CISC.
On the other hand, the F18A doesn't require managing return addresses.
While the indirect call may require more time, the fact that the called subroutine need not have any prolog and/or epilog code may end up with a net savings in time.

### Conclusion

With only 39% of the program space consumed by 18-bit literals, the F18A provides significantly superior code density compared to the S16X4, despite packing fewer instructions per packet.

The F18A's reasonably fast indirect subroutine performance makes it adept at object-oriented, functional, and highly modular software design.

As with the S16X4, explicit exposure of effective address calculations provides opportunities for the programmer to amortize them into near insignificance, particularly if exploiting sequential memory access patterns.
With careful attention to program structure, performance can be weighted closer to RISC levels of performance.


### Listing 1.

```
outsfc:         ds      1

os_bitmap       equ     0
os_fontBase     equ     1
os_flip         equ     2
os_scroll       equ     3
os_x            equ     4
os_y            equ     5
os_ch           equ     6

p:              ds      1
q:              ds      1

plt:            li      p
                sta
                fma

                sta
                fma
                li      q

                sta
                fma
                sta

                sma
                li      p
                sta

                fma
                li      256
                nop
                add

                sma
                li      q
                sma

                fma
                li      80
                nop
                add

                sma
                rfs

pltc:           li      8
                push

L01:            call    plt

                next    L01

                rfs

zerop:          li      outsfc
                lda
                fma

                li      os_fontBase
                nop
                add

                lda
                fma
                li      outsfc

                lda
                fma
                li      os_ch
                nop

                add
                lda
                fma
                nop

                add
                li      p
                lda

                sma
                rfs

zeroq:          li      outsfc
                lda
                fma

                li      os_y
                nop
                add

                lda
                fma
                li      640

                lda
                li      18
                push

                li      0

L02:            nop
                muls
                unext

                drop
                drop
                sta

                li      outsfc
                lda
                fma

                li      os_bitmap
                nop
                add

                lda
                fma
                add

                li      outsfc
                lda
                fma

                li      os_x
                nop
                add

                lda
                fma
                nop
                add

                li      q
                lda
                sma
                rfs

addrs:          call    zerop

                jmp     zeroq

flip:           li      outsfc
                lda
                fma

                li      os_flip
                nop
                add

                lda
                fma
                push
                rfs

crsr:           li      outsfc
                lda
                fma

                li      os_x
                nop
                add

                lda
                fma
                li      -79

                nop
                add
                jpl     L03

                drop
                li      outsfc
                lda

                fma
                li      os_x
                nop
                add

                lda
                fma
                li      1
                nop

                add
                sma
                rfs

L03:            drop
                li      0
                li      outsfc

                lda
                fma
                li      os_x
                nop

                add
                lda
                sma

                li      outsfc
                lda
                fma

                li      os_y
                nop
                add

                lda
                fma
                li      -24
                nop

                add
                jpl     L04
                drop

                li      outsfc
                lda
                fma

                li      os_y
                nop
                add

                lda
                fma
                li      1

                nop
                add
                sma
                rfs

L04:            drop
                li      outsfc
                lda

                fma
                li      os_scroll
                nop
                add

                lda
                fma
                push
                rfs

plotch:         call    addrs

                call    pltc

                call    crsr

                jmp     flip
```

### Listing 2.

This program should behave as listing 1, but has been refactored for maximum code density.

```
outsfc:         ds      1

os_bitmap       equ     0
os_fontBase     equ     1
os_flip         equ     2
os_scroll       equ     3
os_x            equ     4
os_y            equ     5
os_ch           equ     6

p:              ds      1
q:              ds      1

plt:            li      p
                sta
                fma

                sta
                fma
                li      q

                sta
                fma
                sta

                sma
                li      p
                sta

                fma
                li      256
                nop
                add

                sma
                li      q
                sma

                fma
                li      80
                nop
                add

                sma
                rfs

pltc:           li      8
                push

L01:            call    plt

                next    L01

                rfs

_outsfc__:      li      outsfc
                lda
                fma
                nop

                add
                lda
                rfs

_outsfc_os_x:   li      os_x
                jmp     _outsfc__

_outsfc_os_y:   li      os_y
                jmp     _outsfc__

zerop:          li      os_fontBase
                call    _outsfc__

                fma
                li      os_ch
                call    _outsfc__

                fma
                nop
                add

                li      p
                lda
                sma
                rfs

zeroq:          call    _outsfc_os_y

                fma
                li      640
                lda

                li      18
                push
                li      0

L02:            nop
                muls
                unext

                drop
                drop
                sta

                li      os_bitmap
                call    _outsfc__

                fma
                nop
                add

                call    _outsfc_os_x

                fma
                nop
                add

                li      q
                lda
                sma
                rfs

addrs:          call    zerop

                jmp     zeroq

flip:           li      os_flip
                call    _outsfc__

                fma
                push
                rfs

crsr:           call    _outsfc_os_x

                fma
                li      -79
                nop
                add

                jpl     L03
                drop
                fma

                li      1
                nop
                add

                sma
                rfs

L03:            drop
                li      0
                sma

                call    _outsfc_os_y

                fma
                li      -24
                nop
                add

                jpl     L04

                drop
                fma
                li      1

                nop
                add
                sma
                rfs

L04:            drop
                li      os_scroll

                call    _outsfc__

                fma
                push
                rfs

plotch:         call    addrs

                call    pltc

                call    crsr

                jmp     flip
```

