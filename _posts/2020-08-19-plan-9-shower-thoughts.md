---
layout: post
title: "Plan 9 Shower Thoughts"
author: "Samuel A. Falvo II"
---

# Plan 9 Shower Thoughts

As I woke up this morning, I had one of those random thoughts which makes me think, "Huh, why not?"  We have a number of "tiny Unixes" for 8-bit processors of [all](http://www.fuzix.org/) [ilk](http://lng.sourceforge.net/).  While far from POSIX compliant, they all are perhaps surprisingly faithful to early Unix capabilities.  So, this got me thinking: what would a "tiny Plan 9" look like?  I make NO illusions of compatibility with real Plan 9, any more than Lunix is compatible with Linux.  But, even so, it is a fun thought experiment.

For now, I'm considering only smallish 8-bit systems like the RC2014 or Commodore 64.  One of the first and most important tasks for any kernel is memory management.  Plan 9 has a tiny set of system calls for managing memory; however, they still assume a page-based MMU.  `segattach(2)` and `segdetach(2)` can be shoehorned into a flat, single address space under very tightly controlled circumstances.  However, because of the need for scatter-loading binaries, `brk(2)`, `sbrk(2)`, and `segfree(2)` are out of the picture entirely, however.  (For those wondering, segattach(2) would need the virtual address parameter, `va`, to be set to zero at all times.  Further, the `attr` input would be ignored.)

The next task for the kernel is managing running programs.  Nearly all early 8-bit CPUs are built with absolute memory references in mind; the idea of pointer-relative addressing, much less PC-relative, just wasn't "a thing" back in the 70s.  Thus, the kernel will need to load programs using a relocatable binary file format (see my article on my Kestrel blog, [On ELF, Part 2](http://kestrelcomputer.github.io/kestrel/2018/02/01/on-elf-2)).  And, once loaded, those binaries are not moving.

Since this kind of *scatter loading* will all happen in a single address space, `rfork(2)` and `exec(2)` are similarly out of the picture.  An alternative method of launching programs will be necessary.  Also, `segattach(2)` and `segdetach(2)` will need to be vigilent about minimizing address space fragmentation.  Quite doable, but at some cost in implementation complexity.

All the other system calls should be doable, however.  IPC *could* be handled via anonymous pipes, just like it's done in Plan 9 currently (pipe buffers would need to be smaller than 4KB though!).  If we were to stop here, I'd reckon you'd need a minimum of a 96KB RAM system to run everything (32KB for pipe buffers, 64KB for kernel and small set of processes) to make a halfway usable system.  Obviously, more RAM is better; *most* of your overhead will be in IPC buffering.  A C128 could be a good platform for this configuration.

However, if we're willing to bend what it means to be a "Plan 9", it might be more appropriate to use a system of callbacks instead, since these don't use any buffers beyond what the client prepares ahead of time.  A system similar to VMS' Asynchronous System Traps or L4's synchronous message passing might perhaps be used to implement this.  But, now we're getting deeper into microkernel territory.

So, for example, if process A calls open("/aMountPt/foo"), it'd result in a system call which looks up /aMountPt/foo, sees that it doesn't exist, then tries /aMountPt, sees that it *does* exist, and schedules a call to that mount point's "open" callback.  It must be scheduled since it must run in the context of another process.  You could use a system of "migrating threads", but now we're veering well off both the Plan 9 AND microkernel paths, and has a whole bunch of its own problems.

For systems with more advanced memory management, such as a 65816-based system with a PMMU driving the ABORT# pin, or a dual-CPU system where one CPU is the "user CPU" and the other the "supervisor CPU", it's possible that `rfork(2)`, `exec(2)`, and the more complete set of segment functionality can be made available again.  Since pages are not very granular, however, you'd probably need a system with at least 1MB to 2MB of RAM before you end up with a usable foundation.

So, can a tiny Plan 9 be constructed for small systems?  I don't know for sure; I've never actually tried.  However, I *think* it can as long as you're willing to let go of those system facilities which have a hard dependency upon a PMMU.  Such a system would look like, at a minimum, a Commodore 64 equipped with an off-line bank of RAM for use as pipe buffers.  More likely than not, though, as not every system as readily available RAM expanders like the Commodore REU, you'd also need to let go of the use of pipes all-together as your go-to choice for IPC, and start relying on asynchronous system traps or upon microkernel-like synchronous message queues as your IPC mechanism.

