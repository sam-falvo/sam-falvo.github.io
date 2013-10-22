---
layout: post
title: "How Does MISC Stand Up to RISC, CISC? (1/7)"
author: Samuel A. Falvo II
publish: true
---

We often hear anecdotes about how RISC so deftly outperforms traditional CISC architectures.
(Indeed, it's important to qualify, for most CISC architectures today are really RISC machines in disguise.)
But, if you're involved with Forth at all, you've almost certainly also heard of an architecture style called MISC.
No doubt, you've probably heard how awesome MISC compares to RISC.
However, I've never seen an objective comparison between CISC, MISC, and RISC architecture styles from the software perspective.

I try to put these processor architectures into perspective using real-world experience with some processors I've used in the past.
This article introduces my methodology for accomplishing this.
The next article will dive into the S16X4 MISC architecture.

<!-- more -->

### Features I'm Interested In Comparing

Efficient expression evaluation almost goes without saying, yet it isn't the only thing one can do with a processor.
Without the ability to retrieve data from memory and store data back, expression evaluation serves no purpose.
Modern software design relies heavily on *records*, each of which contains *fields*, which a program may randomly access.
Thus, we want to specifically exercise this method of information storage and retrieval.
However, even information access and its subsequent evaluation no longer cuts it for, e.g., desktop- or server-grade software architecture.
Today, a good architecture will naturally support polymorphic *interfaces* as well.

Everywhere you look, test-driven development techniques aids in high-quality software production.
Its benefits, being one of process and not of technology, transcends both the choice of processor and the choice of programming language.
Software produced in this way tends to heavily depend on polymorphic interfaces to provide linkage to other components,
typically written independently and not necessarily in the order of dependency.
Explicit processor support for polymorphism, in the form of fast, indirect subroutine calls,
aids in writing independently testable chunks of code that still runs fast in production environments.

### Program Description

To provide a consistent basis for comparison,
I'm going to implement programs for several processors, so as to demonstrate expression evaluation,
access to parameters found in control blocks pointed to through a single pointer,
and calling services through explicit interfaces.
The problem to solve involves placing text glyphs on a bitmap at some current cursor position, then advancing the cursor.
I'll write the software in such a way as to not depend on any particular OS or hardware feature.
Thus, access to system-level services must occur through an (polymorphic) interface.

The listing below contains the basic program, written in high-level Forth.
It contains an mix of indirect subroutine calls, expression evaluation, and structure access sufficient for comparison purposes. 
Since MISC architectures (so far) always implement stack architectures, I assume the reader understands Forth.
However, just in case you do not, I also list the closest C code equivalent to serve as an understanding aid.

Listing 1.

```
\ Forth Program

variable outsfc                  \ pointer to an OutputSurface structure

0 cells constant os_bitmap       \ pointer to bitmap memory
1 cells constant os_fontBase     \ pointer to font memory
2 cells constant os_flip         \ pointer to function to display bitmap
3 cells constant os_scroll       \ pointer to function to scroll bitmap
4 cells constant os_x            \ where to put char (0 <= x < 80)
5 cells constant os_y            \ where to put char (0 <= y < 25)
6 cells constant os_ch           \ Character to display (0..255)

variable p
variable q

\ Plot character by moving font data into the bitmap.
: plt      p @ c@ q @ c!  256 p +!  80 q +! ;
: pltc     8 for plt next ;

\ Configure p and q temporaries to point to the correct values prior to pasting
\ the character.
: 0p       outsfc @ os_fontBase + @  outsfc @ os_ch + @ +  p ! ;
: 0q       outsfc @ os_y + @ 640 *  outsfc @ os_bitmap + @  +  outsfc @ os_x + @ +
           q ! ;
: addrs    0p 0q ;

\ After drawing, we "flip" the display to render it to the user's display.
: flip     outsfc @ os_flip + @ execute ;

\ Crsr updates the cursor position on your behalf.  If necessary, it will
\ scroll as well.
: crsr     outsfc @ os_x + @ 79 u< if 1 outsfc @ os_x + +! exit then
           0 outsfc @ os_x + !
           outsfc @ os_y + @ 24 u< if 1 outsfc @ os_y + +! exit then
           outsfc @ os_scroll + @ execute ;

\ Public entry point.
: plotch   addrs pltc crsr flip ;
```

Listing 2.

```c
/* Equivalent C program */

typedef struct OutputSurface {
    char   *bitmap;
    char   *fontBase;
    void   (*flip)();
    void   (*scroll)();
    int    x;
    int    y;
    char   ch;
} OutputSurface;

OutputSurface *outsfc;

static char *p, *q;

void plotch() {
    p = outsfc->fontBase + outsfc->ch;
    q = outsfc->bitmap + outsfc->y * 640 + outsfc->x;
    for(int i = 0; i < 8; i++) {
        *q = *p;
        p += 256;
        q += 80;
    }
    if(outsfc->x < 79) {
        outsfc->x++;
        outsfc->flip();
        return;
    }
    outsfc->x = 0;
    if(outsfc->y < 24) {
        outsfc->y++;
        outsfc->flip();
        return;
    }
    outsfc->scroll();
    outsfc->flip();
}
```

### Selection of Processors

I will compare microprocessors that I've direct and/or relatively recent experience in each of the architecture styles.
In the MISC category, I'll include the S16X4 (a MISC of my own design, powering my Kestrel-2 computer) and the F18A MISC core designed by Chuck Moore himself.
Because of how reliable and predictable MISC architectures are, I'll make regular comparisons to an as-yet unimplemented CPU intended for the Kestrel-3, the eP64.

In the CISC category, I'll showcase the Western Design 65816 microprocessor running in 16-bit native mode, and
the Motorola 68000 microprocessor, at least as used in the Commodore-Amiga 500.
Other variants of the 68000 exist these days, but as I've not used those, I cannot guarantee their timing remains consistent with the original design.

I've the least experience with overtly RISC processors, but I once worked with MIPS R3000-based hardware back at Hifn, a long time ago.
I'll try to muddle my way through trying to remember MIPS mnemonics and delay slot rules.
However, if I remember rightly, instructions for each processor behaved very predictably.
In the case of the R3000, provided no pipeline stalls, it retired instructions at a rate of 1 cycle per evaluating instruction, and 2 cycles for loads and stores.
If someone with more recent MIPS R3000 experience finds errors, please report them to me via Github issues.
I'll happily revise this article with verifiable corrections.

While I have used PowerPC and ARM devices, I've never written assembly-language for these platforms.
Nonetheless, observations made about the R3000 can apply generally to other RISC designs as a first-degree approximation of expected performance.

For every processor with a cache, I'm explicitly assuming *no* cache misses.

### Coming Up Next

This article is already getting a bit long, so I'm going to cut it here.
In the next article, I provide a translation of the above program into the S16X4 MISC assembly language, and provide a simple analysis of its code.

