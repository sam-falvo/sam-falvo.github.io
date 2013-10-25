---
layout: post
title: "How Does MISC Stand Up to RISC, CISC? (2/7)"
author: Samuel A. Falvo II
publish: true
---

I have yet to see objective comparisons between MISC, RISC, and CISC architectures.
MISC processors often get a bad rap for their stack-based micro-architectures; however, generally-available evidence to support these negative views seems lacking.
I try to put these processor architectures into perspective using real-world experience with some processors I've used directly in the past.
I hope the data revealed will help others in answering questions on which architecture proves right for them.

In this article, I highlight the S16X4 processor, a design which I built for the Kestrel-2 home-made computer.
Details of this processor may be found in [its datasheet](https://github.com/sam-falvo/kestrel/blob/master/cores/S16X4/doc/datasheet.pdf).

<!-- more -->

### Space Consumption

The S16X4 derives from the Steamer16, a word-address machine capable of 16-bit wide words.
The S16X4 includes instructions for addressing memory with byte-granularity,
which I exploit in our example program for the purposes of plotting an 8x8 glyph onto a bitmap.
However, this does not materially affect program size.
Even were I to stick with pure word addressing, I would just say that the font data is packed with two bits per pixel.
This results in a program with exactly the same number of instructions to execute.

Different routines in the program have been sized:

```
Name       Size (%)
====================
plt         16 9.8%
pltc        18 11%
zerop (0p)   8 4.8%
zeroq (0q)  14 8.5%
addrs        8 4.9%
crsr        33 20%
plotch      15 9.1%
setOutSrc   52 32%
====================
TOTAL      164 words (328 bytes)
====================
```

I count 102 uses of the `li` instruction, which means 102 of the 164 words contains literal information, much of it repeated.
With a richer instruction set, one could find opportunities to reduce the dependency on the `li` instruction, thus helping to improve code-density.

### Runtime Performance

Without exception, *all* S16X4 instructions takes one clock cycle to execute.
Additionally, the CPU further takes an additional clock cycle to fetch a batch of four instructions.
These simple rules makes estimating run-time performance easy, as the following table illustrates.

```
Name       Cycles (%)
====================
plt         29+(0*0)
pltc        11+(8*(16+plt))
zerop (0p)  15+(0*0)
zeroq (0q)  27+(0*0)
addrs       12+(0*0)+0p+0q
crsr        55+(0*0)
plotch      22+(0*0)+addrs+crsr+pltc
setOutSrc   95+(0*0)
====================
TOTAL      266+(344) = 610 cycles
====================
```

This table tells us that initializing a bitmap and printing a single character takes 610 cycles to complete, minimum.
Additional characters may be plotted with only 515 cycles, for no need exists to invoke `setOutSrc` for each character intended.

The `pltc` procedure requires the lion's share of this time, taking 371 cycles on its own (11+(8*(16+29))).
`plt` takes seven cycles to transfer a single byte from the font bitmap to the destination bitmap.
It takes a further seven cycles to increment `p`, and another seven for `q`.
Within this routine, at least, the S16X4 compares with a classic CISC processor equipped with many general-purpose registers, such as the MC68000.

### Cost of Structure Field Access

Accessing fields off of a base pointer takes seven cycles and four words to complete:

1.  Fetch the `li base; fwm; li field_offset; add` bundle.
2.  Spend four cycles executing these instructions.
3.  Fetch the `fwm` in the next bundle.
4.  Spend one cycle executing this instruction.

Accessing an absolute memory location takes two cycles on average, three worst-case, while requiring half as many words of memory to encode.
Clearly, executing references to absolute memory locations proves much faster, and as well, consumes far less program memory.

If left as-is, the S16X4 MISC would behave not entirely unlike a Motorola 68000 when reading or writing structure fields.
I've optimized the software in listing 1 to predominantly access absolute memory locations.
To ensure these absolute locations always reflect the caller's choice of OutputSurface, the caller must invoke `setOutSfc` before using `plotch` to render text.
Developers of MISC software will recognize this style of software design as more idiomatic,
largely for the aforementioned performance and program size reasons.

Observe that outsfc pointer never changes while it's in normal use by either the calling or called subprogram.
`setOutSfc` works like a cache refill operation --- it writes fields that change back to original structure, then caches fields from new structure into absolute locations.
Caching, then, amortizes the cost of prefetching fields across all absolute memory references corresopnding to a field access.

It turns out, as shown in listing 1, the program makes 15 field accesses for each call to `plotch`.
The `setOutSfc` routine consumes 95 cycles, assuming it writes back as well.
Thus, when writing a character, those 95 cycles virtually "spread out" over the 15 field accesses.
This corresponds to (95/15)=6.333 cycles per field access.
Since each absolute reference averages two cycles, the `plotch` routine runs _as if_ each field access consumed 2+6.333 = 8.333 cycles.

It turns out, caching holds no value when plotting only a single character on the bitmap.
However, when repeatedly invoking `plotch`, e.g., when printing a _string_ of characters, overhead drops very rapidly.
Printing the text, "Hello", involves only five characters.
Yet, since the overhead for `setOutSfc` remains constant, amortized overhead becomes (95/(5*15))=1.267 cycles,
such that `plotch` behaves as if each field access takes only 3.267 cycles.
If printing the average-sized line of text, say 40 characters, the overhead drops further to sub-unity: 0.158 cycles!
If printing an average-sized screen, say in the ballpark of 1000 characters for an 80x25 bitmap, overhead drops to insignificance: 0.00633 cycles per field access.
The less frequently a program calls `setOutSfc`, the closer to two cycles each field access becomes.

Therefore, even though we needed to alter our strategy a bit to realize its benefits,
the S16X4 MISC processor competes favorably against very sophisticated CISC architectures like the MC68000 for structure field references in the best case,
and equals their performance in the pathologically worst case.
The MIPS R3000 still performs better due to its dependency on pipelining, but only by a factor of two in the long term.

### Cost of Indirect Subroutine Call

The S16X4 lacks _any_ kind of subroutine mechanism in hardware.
Thus, a programmer or compiler must synthesize this capability from existing instructions and conventions.
Idiomatically, the S16X4 run-time environment stores return addresses in absolute memory locations, to save time.
Of course, this prevents re-entrancy and recursion without more complex prolog and epilog procedures.

A normal subroutine, then, involves a code sequence such as:

```
li      return_address
li      subroutine_address
go
```

This takes four cycles to complete, not including latency introduced by the subroutine's prolog and epilog.
In this case, the S16X4 falls between CISC and RISC in subroutine performance.

Because the programmer must synthesize subroutine calls of all kinds, supporting indirect calls comes naturally and easily.

```
li      return_address
li      pointer_to_subr_address
fwm
go
```

Note the simple introduction of a fetch instruction just before the go opcode.
This adds a cycle to the operation.

In this case, the S16X4 runs closer to RISC speeds than classic CISC.
Indeed, the 65C816 and MC68000 both take something on the order of 20 clock cycles to accomplish the same task.
(This will be revealed in several articles.)

### Conclusion

With 62% of the program space consumed by 16-bit literals, the S16X4 provides a code density on par with a 32-bit RISC.

I was surprised to find that the S16X4, despite lacking overt support for structured data access, potentially performs better at that task than it does moving bytes into a bitmap,
_provided_ one adopts some means of amortizing the effective address calculations, such as with field caching.
If one cannot avoid the effective address calculations (e.g., as with creating or destroying a recursive subroutine's activation frame), performance will _never_ be worse than a classic CISC.

The S16X4's unusually high indirect subroutine performance makes it surprisingly adept at object-oriented, functional, and highly modular software design.

The S16X4, despite having only 12 instructions, remains a surprisingly powerful MISC processor that offers performance somewhere in between classic CISC and RISC.
Explicit exposure of effective address calculations provides opportunities for the programmer to amortize them into near insignificance.
With careful attention to program structure, performance can be weighted closer to RISC levels of performance.


### Listing 1.

```
outsfc:         .word   0

os_bitmap       equ     0
os_fontBase     equ     2
os_flip         equ     4
os_scroll       equ     6
os_x            equ     8
os_y            equ     10
os_ch           equ     12

p:              .word   0
q:              .word   0

                ; Extra storage for temporaries, etc.
plt_pc:         .word   0
pltc_pc:        .word   0
zerop_pc:       .word   0
zeroq_pc:       .word   0
addrs_pc:       .word   0
plotch_pc:      .word   0
setOutSfc_pc:   .word   0

t0:             .word   0
outsfcNew:      .word   0

                ; Cached fields of current OutputSurface.
cc_bitmap:      .word   0
cc_fontBase:    .word   0
cc_flip:        .word   0
cc_scroll:      .word   0
cc_x:           .word   0
cc_y:           .word   0
cc_ch:          .word   0


plt:            li      plt_pc
                swm
                li      p
                fwm

                fbm
                li      q
                fwm
                sbm
                
                li      p
                fwm
                li      256
                add

                li      p
                swm
                li      q
                fwm

                li      80
                add
                li      q
                swm

                li      plt_pc
                fwm
                go

pltc:           li      pltc_pc
                swm
                li      8
                li      t0

                swm

pltc_l1:        li      pltc_l2
                li      plt
                go

pltc_l2:        li      t0
                fwm
                li      -1
                add

                li      t0
                swm
                li      t0
                fwm

                li      pltc_l1
                nzgo
                li      pltc_pc
                fwm

                go

zerop:          li      zerop_pc
                swm
                li      cc_fontBase
                fwm

                li      cc_ch
                fwm
                add
                li      p

                swm
                li      zerop_pc
                fwm
                go

                xref    mulBy640
zeroq:          li      zeroq_pc
                swm
                li      cc_y
                fwm

                li      cc_y
                fwm
                add
                li      mulBy640

                add
                fwm
                li      cc_bitmap
                fwm

                add
                li      cc_x
                fwm
                add

                li      q
                swm
                li      zeroq_pc
                fwm

                go

addrs:          li      addrs_pc
                swm
                li      addrs_l1
                li      zerop

                go

addrs_l1:       li      addrs_pc
                fwm
                li      zeroq
                go

crsr:           li      crsr_pc
                swm
                li      cc_x
                fwm

                li      -79
                add
                li      $8000
                and

                li      crsr_l1
                zgo
                li      cc_x
                fwm

                li      1
                add
                li      cc_x
                swm

                li      crsr_pc
                fwm
                go

crsr_l1:        li      0
                li      cc_x
                swm
                li      cc_y

                fwm
                li      -24
                add
                li      $8000

                and
                li      crsr_l2
                zgo
                li      cc_y

                fwm
                li      1
                add
                li      cc_y

                swm

crsr_l3:        li      crsr_pc
                fwm
                go

crsr_l2:        li      crsr_l3
                li      cc_scroll
                fwm
                go

                xdef    plotch
plotch:         li      plotch_pc
                swm
                li      plotch_l1
                li      addrs

                go

plotch_l1:      li      plotch_l2
                li      pltc
                go

plotch_l2:      li      plotch_l3
                li      crsr
                go

plotch_l3:      li      plotch_pc
                fwm
                li      cc_flip
                fwm

                go

                xdef    setOutSfc
setOutSfc:      li      setOutSfc_pc
                swm
                li      outsfc
                fwm

                li      setOutSfc_l1
                zgo
                li      cc_x
                fwm

                li      outsfc
                fwm
                li      os_x
                add

                swm
                li      cc_y
                fwm
                li      outsfc

                fwm
                li      os_y
                add
                swm

setOutSfc_l1:   li      outsfcNew
                fwm
                li      outsfc
                swm

                li      outsfc
                fwm
                li      os_bitmap
                add

                fwm
                li      cc_bitmap
                swm
                li      outsfc

                fwm
                li      os_fontBase
                add
                fwm

                li      cc_fontBase
                swm
                li      outsfc
                fwm

                li      os_flip
                add
                fwm
                li      cc_flip

                swm
                li      outsfc
                fwm
                li      os_scroll

                add
                fwm
                li      cc_scroll
                swm

                li      outsfc
                fwm
                li      os_x
                add

                fwm
                li      cc_x
                swm
                li      outsfc

                fwm
                li      os_y
                add
                fwm

                li      cc_y
                swm
                li      outsfc
                fwm

                li      os_ch
                add
                fwm
                li      cc_ch

                swm
                li      setOutSfc_pc
                fwm
                go
```

