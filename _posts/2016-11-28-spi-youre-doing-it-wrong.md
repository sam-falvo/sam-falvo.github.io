---
layout: post
title: "SPI: You're Doing It Wrong"
author: "Samuel A. Falvo II"
---

# SPI: You're Doing It Wrong

Are you planning on using a serial interconnect to replace a large number of wires
with a smaller, more manageable number in a custom peripheral or microcontroller project?
If so, you were probably about to do it completely wrong.

Oh, don't get *me* wrong;
it probably would have been right enough to work for your needs.
However, I still encourage you to think carefully about the *semantics* of such an interface.
As soon as you need to multiplex an amount of data greater than the number of wires you have available to you,
you introduce the need for (semi-)intelligent endpoints, and a protocol that these endpoints must understand.
In this article, I'll talk about SPI in particular; but,
be aware that it's generally applicable to any narrow interconnect.

Based on my own research idly Googling for articles concerning SPI,
it seems like an awful lot of people run into situations where
[the SPI slave just locks up.](https://www.google.com/search?client=ubuntu&channel=fs&q=SPI+locking+up&ie=utf-8&oe=utf-8)
We need to remember that
[concurrent programming is hard](https://www.cs.cmu.edu/afs/cs/academic/class/15213-s13/www/lectures/23-concurrent-programming.pdf),
and even worse, extremely difficult to prove correct even with formal and automated provers.
I have a hunch
that this is due to people fundamentally misunderstanding the role that SPI plays in their design.

## Understand the Role of SPI

The whole point of your exercise is undoubtedly to save money in your embedded (or not so embedded) design.
Reducing the amount of wires in an interconnect not only reduces PCB space needed to route traffic,
but it also reduces the cost of the packages you need to place on the PCB as well.
In some cases, it even lets you transmit data faster than you could with a parallel bus due to secondary effects of
reducing capacitance, skew, and easier transmission line construction.
Copper traces on a printed circuit board do not fail except under extreme physical duress,
so how do you implement the SPI interconnect to be at least as robust as those wires you're replacing?

**The interconnect should remain functional even if the peripheral it's attached to is not.**

I can't re-iterate this enough:
the purpose of SPI is *not* to just route message-oriented traffic to other chips.
You can do that with a wider bus just as well,
and do so faster and with less burden on the software in your controllers in the process.

Its *actual* purpose is to emulate the presence of a wider interconnect.
This is a fundamental principle which I see routinely violated,
both in simple hacked projects and, occasionally, in commercial vendors alike.

## Recap: How SPI Works

SPI works on exactly two principles:

1.  SPI exposes a strict, master/slave relationship.  When the master talks, the slave *must* listen.  There can be no exception to this rule.  Note the absence of feedback signals like interrupt requests, retries, bus errors, and the like.  Some devices have them, but they're not universally supported, and their meaning often changes from device to device.  The only constant is MOSI, MISO, SS#, and CLK.  That's *all* you get to depend on.

2.  SPI peripherals expects you to adhere to a strict request/response protocol.  The slave can communicate only while the master is sending something.  This means, after the master issues its command to the slave, the master must [busy-wait](https://en.wikipedia.org/wiki/Busy_waiting) on the device while it's busy formulating a response.  As a performance optimization (which is really an accident of SPI's implementation), a master *may* send another command while it's receiving a reply to a former request, or it *may* even queue multiple commands if the slave allows for it.  Again, not all slaves support such sophisticated protocols, and if it does, not all commands may be supported in a common manner.  Consider: an SD card will allow a master send a command to terminate an in-progress block-read operation, but how will it react if you try to queue a block-write command?

The simplest possible SPI interconnection between a master and slave looks, schematically, something like this:

         Master                            Slave
         +--+--+--+--+--+--+--+--+         +--+--+--+--+--+--+--+--+
    +--->|  |  |  |  |  |  |  |  |-------->|  |  |  |  |  |  |  |  |----+
    |    +--+--+--+--+--+--+--+--+         +--+--+--+--+--+--+--+--+    |
    |                                                                   |
    +-------------------------------------------------------------------+

As bits are shifted out of the master, they're shifted into the slave.
At the same time, bits from the slave are shifted into the master.
This arrangement allows for serial communications between peers
with about half of the resources needed by even the smallest UART.
Recall that a UART requires an independently functioning transmit and receive shift register,
so even the smallest of UARTs will require four registers between both peers.

When the slave has something to send to the master,
it loads its transceiver register with a byte, so that
the next time the master tries to send something, it receives that data.
The key word here is **when**;
what happens if your slave is busy waiting on something of its own?
It can be a while before it responds;
the master has no idea if the slave is OK during this time.
Indeed, the slave could well be *locked up completely*,
in which case the master will wait indefinitely (at least until it times out).

## Isolation

Just as a big bunch of wires connecting a peripheral ought to never fail,
so too do you not want your tiny bundle of wires that *replaces* said big bunch of wires to fail either.
SPI is a replacement for a big bunch of wires,
not for what travels over those wires.
That *directly implies* that whatever is responsible for handling the SPI slave interface must never fail, either.

### Controller-based Isolation

The simplest method to achieve this goal of autonomy
is to just dedicate an entire microcontroller just to the interconnect.
The programming on this microcontroller sits in a tight loop, constantly monitoring the SPI link for activity,
and upon receiving commands, formulates responses as quickly as possible.
The actual *function* of the peripheral is none of the SPI controller's concern;
it's merely a *relay* for reading and writing data, command and control, and telemetry.

    +-----+  |  +-----+   +-----+
    |     |<--->| SPI |<->|     |
    |     |  |  +-----+   |     |
    |     |  |            |     |
    +-----+  |            +-----+
    Master   |  Slave

This is where most projects that use SPI seem to fall down.
They tightly integrate reacting to SPI events into their project's main loop.
This is dangerous, for precisely the same reasons that
[cooperative multitasking](https://en.wikipedia.org/wiki/Cooperative_multitasking)
is dangerous on general purpose computers.
It takes only one bad actor to derail the project's main event loop, and when it happens, you lose everything.

In contrast, a project with a dedicated interconnect controller has an additional level of safety.
Look at the failure modes of a project with a split controller:

|SPI Controller|Function Controller|Perceived Failure|Remediation|
|:------------:|:-----------------:|:----------------|:----------|
|Working.      |Working.           |None.            |None.      |
|Working.      |Seized.            |Status flags and performance counters report no activity from the function controller in a reasonable period of time.|Master can issue reset command(s) in an attempt to interrupt the function controller, or perhaps to even reboot it via its reset pin directly.|
|Seized.       |Does not matter.   |No response from the SPI controller for even the most basic of queries or commands.|None; this is likely a power outage, disconnected cable, or unseated/missing/under-powered chip.|

With a separate SPI microcontroller,
you can at least send a command via SPI to reset the peripheral controller as a means of attempting to regain control.
Depending on your design, you might even have independent control over what gets reset or re-initialized.
With a unified microcontroller, you have no way of telling what's going on.
Anyone who has ever suffered protocol errors with the SD protocol
knows full well just how prone those things are to seizing hard,
and with no lifesigns until the card is pulled out and re-inserted.
Or until the master *emulates* this by power-cycling the card with dedicated circuitry for that task.
But I digress; point is, SD cards are, in my experience, guilty of using a single controller for SPI and protocol control,
and as a result, are fickle in the face of errors.

### Process-based Isolation

However, if you're so cost conscious that you cannot afford two separate controllers,
then you should consider instead using at least two *processes* running on the same controller.
Here, I define *process* in the same way Erlang might:
a domain of protection intended to isolate one program from another,
where no two processes share state, and
only communicate through a well-trusted signaling or message passing system.
For mid-grade microcontrollers,
it's unlikely you'll have memory management units to help enforce this,
so you will need to resort to a *prioritized, preemptively* scheduled multi-threading kernel with carefully written code,
manually making sure that your SPI driver *never* touches any other task's memory, and vice versa.
Clearly, this depends on a lot of "what-ifs",
so it's clearly the least provable method to achieve isolation.
But, with a careful choice in programming languages, you can probably pull this off without much trouble.

A better solution would be to use a larger microcontroller that has a hardware memory management unit (MMU) of some kind.
This does not have to be a paging MMU, either.
For example, the current
[RISC-V privileged instruction set specification](https://riscv.org/wp-content/plugins/pdf-viewer/stable/web/viewer.html?file=https://content.riscv.org/wp-content/uploads/2016/11/riscv-privileged-v1.9.1.pdf#page=54&zoom=auto,-16,535)
provides support for what's called *base and bounds* protection.
(Essentially a crude form of segmentation.)
This is plenty sufficient to achieve the desired outcome.
Alternatively, some PowerPC- or POWER-based microcontrollers provide special
Block Address Translation, or BAT, registers which could be used here.

### Comparing Controllers vs Processes

With a dedicated controller, you have a dedicated processor tending to the I/O channel.
This means, plainly, that you never have complex timing relationships between what transactions come or go on the link.
With processes, however, you need to make sure that the SPI driver process is scheduled in hard real-time,
so as to avoid overrun errors in the SPI data register/FIFO.
For that matter, you also need to make sure the process is scheduled when you have a response to send as well.
You don't want to risk the master timing out due to jitter in how your OS schedules your driver thread.
This is why I emphasized using a *prioritized, preemptively* scheduled kernel above.
A cooperatively scheduled kernel,
prioritized or not,
will just put you right back in the same situation as a naive implementation.
It's *doable*, but only if you can statically prove timing closure for all possible inputs,
*including those not anticipated in the field.*

On the other hand, a dedicated controller will cost you printed circuit board space.
If it's one thing I've noticed over the years,
it's that PCBs are surprisingly expensive.
When PCB area starts to dominate the cost of adding an additional MCU cost,
you might want to consider upgrading to a more powerful MCU with memory protection, and relying on processes instead.
There's also the cost of coupling the SPI controller to the function controller as well.

## Turtles All the Way Down

Above, I mentioned that the purpose of SPI is to emulate a wider interconnect.
Typically, wider interconnects have only a small number of "commands."
Meanwhile, narrower interconnects tend to have a bewildering array of commands.
Count the number of commands defined by the MMC/SD interface standards.
I'll wait.

Here are the commands exposed by the 6502 microprocessor, for example:

* Read byte at address A15-A0.
* Write byte D7-D0 at address A15-A0.
* Fetch opcode D7-D0 at address A15-A0.  Sample interrupts.

That's it.
The rest of the CPU's interface is, in some form,
a means of controlling how fast the CPU interacts with the addressed peripheral.

So, here's a question:
if we were to build a 6502 emulator that runs over an SPI interface of some kind,
thus letting us replace a 40-pin parallel interconnect with a 6-pin PMOD interface,
do we encode a series of higher-level messages like, "Fetch bytes starting at address",
or do we remain faithful to the original parallel interconnect?

There are, of course, arguments in favor of both methods; but,
I'm going to argue that you should *stay faithful to the original interconnect* as a basis.
You can always add enhanced functionality later if it's warranted.

Sticking with the original interface semantics has several benefits.
First and foremost, a 6502 doesn't care if a RAM or I/O controller responds to a read request.
It'll happily read garbage if nothing's there to respond to the data request.
I argue that the SPI link's command set *must* react in the same manner.

How would one handle a slow peripheral?
SPI devices tend to send $FF as an idle character,
so the master would busy-wait on the slave until it received some byte *other* than $FF.
Then it would know the next several bytes contains the response to the previous command.
In this case, a single byte containing the value requested.

We can't wait forever, however, and the SPI slave controller knows this.
There are two ways of handling this situation:

1.  The master *negates* SS# when wants to cancel the command in progress.  This causes the slave interface to abort its wait on results, and probably should also notify the function controller to cancel its pending operation as well.  This frees the slave interface up to respond to another SPI command.

2.  The slave interface controller *times out* itself (remember, it's logically independent of the actual function controller!), and sends back an error response to the master, which can decide to re-issue the command or do something else later.

Either one of these approaches is OK by me.
The first option lets the master control its own timing parameters to some extent,
while option 2 allows for simpler master software stacks.
I should point out that options 1 and 2 are not mutually exclusive;
in fact, I'd go further to say that option 1 isn't even an option;
it really is the minimum functionality expected of an SPI link.

This brings us to the whole concept of *layering* protocols within protocols.
It's how the Internet works, for instance, and it can work for SPI as well.
The command set supported by the SPI slave interface should *terminate* at the SPI slave interface.
Requests for the function being controlled should appear *inside* the commands ferried to the SPI slave.

For instance, if I were to redesign the SD protocol from scratch,
I would have to insist on using commands not entirely dissimilar to what IBM mainframes use when talking to disk units:

* Read N sectors of data.
* Write N sectors of data.
* Receive the next N bytes of command data, and have the function execute it.
* Sense up to N bytes of status information from the function.
* Is the function alive?  Alternatively, report the latest metric M of the function.
* Reset the function.

(In point of fact, I strongly urge readers of this article
to study the the
[IBM z/Architecture Principles of Operation](https://www.google.com/search?client=ubuntu&channel=fs&q=z%2FArchitecture+principles+of+operation&ie=utf-8&oe=utf-8),
in particular the section on how Channel I/O works.)

So, to read a 1024-byte block starting at sector 1234,
I would first "command" the card to seek to sector 1234,
then send the "read" command for 1024 bytes,
then receive the data requested.
If, for some reason this transaction failed, I can use the "is alive?" function to determine health,
and if it's found to not be in a working state,
I can issue a reset function in an attempt to reset it and try again.

Likewise, to determine card capacity, I would first:
"command" the card to produce card capacity, then
"sense" the results back.
Note the differences between command and write:
the former works on bytes and targets the function controller itself,
while the latter works on whole sectors, and targets raw storage.
Likewise with sense versus read.
But all the while, notice that instructions like read, write, command, and sense
*target the SPI slave controller itself.*

## Conclusion

I'd like to conclude by recapitulating some key take-aways from this article:

1.  SPI (or *any* serial interconnect) only proposes to replace a wider interconnect; it *does not* propose to serve any higher-level purpose than that.
2.  SPI interconnects should never fail except in cases where a wider interconnect would also be expected to fail.
3.  Functional units should be designed to expose basic telemetry to the master, so the master can gauge health of the slave as a whole.
4.  SPI slave controllers (or their software equivalents) should not be involved with interpreting the commands intended for the function.
5.  Commands intended for the function should be *encapsulated* inside of commands intended for the SPI slave controller, just as TCP is encapsulated inside of IP packets.  Similarly with responses.
6.  The command set supported by the SPI slave interface should be small enough that you can *easily* prove correctness *without* an automated theorem prover.  Build up from there.

So if you're planning on building an SPI slave,
please keep the above points in mind.
Now go forth and build something not just awesome,
but reliable too!

