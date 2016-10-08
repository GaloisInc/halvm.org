---
layout: post
title: "HaLVM v3: The Vision, The Plan"
date:  2016-04-29 16:00:00 -0700
author: awick
categories: article
---

Who are you, and why are you writing this?
==========================================

Hello! Over the course of the last ten years, I have been writing about, talking
about, and maintaining the Haskell Lightweight Virtual Machine, or HaLVM. The
HaLVM is our [Haskell unikernel](https://en.wikipedia.org/wiki/Unikernel), which
allows us to run Haskell code on "bare metal."

Over the past 6--12 months, I have been receiving enquiries regarding the state
and the future of the HaLVM. How stable is it? What have I
([Adam](https://galois.com/team/adam-wick/)) or we ([Galois](http://galois.com))
been using it for? Does it support feature *X*? Will it ever support ARM? KVM?
VMWare? EC2? Rackspace? What about multi-core, GPU-assisted, PowerPC chips
deployed on rad-hardened satellites with deep software error correction
concerns, spat out as the result of a combined [Coq](https://coq.inria.fr/) and
[Idris](http://www.idris-lang.org/) compilation stream?

This document is an attempt to answer these questions. I will begin by briefly
describing the history of the HaLVM and its current uses. This overview should
provide the reader some background on why some of the "completely batshit crazy"
stuff in the HaLVM is actually well thought out and careful crafted. Or at least
somewhat reasonable.

Just in case you want to skip ahead, though:

  * After discussing the history of the HaLVM, we discuss [what we think is
    wrong with HaLVM 2.0](#wrongwith2).
  * Then we get to our [overview of HaLVM v3.0](#halvm3), and the major
    improvements we have planned for said.
  * Next, we talk [tactics](#tactics). If you want to help out with the HaLVM,
    the previous sections should provide you some context about where we're
    going and why, but this section will point you to specific tasks, places
    in the source code, and forums where we're looking for help.
  * Finally, some realism and acknowledgments in our [final thoughts](#ack).


Of course, if you want to start with a deep technical dive into what makes the
the current version of the HaLVM different from stock GHC, we [now have that
document
available](https://github.com/GaloisInc/HaLVM/wiki/What-is-the-Difference-Between-GHC-and-the-HaLVM%3F). 

A Brief History of the HaLVM
============================

In 2004 or 2005, [Galois](http://galois.com)[^galconn] was hired to help
investigate the design of a trustworthy operating system, based upon a
[microkernel architecture](https://en.wikipedia.org/wiki/Microkernel), with Xen
as the "microkernel" of choice. One of the difficulties of such projects is
creating a design that not only looks good on paper, but also (a) boots and (b)
performs adequately. Like [The
Dude](https://en.wikipedia.org/wiki/The_Big_Lebowski) says, this is a
complicated [problem]: lotta ins, lotta outs, lotta of what have yous.

[^galconn]: Technically we were still Galois Connections at that point. Helping
    category theorists find each other since 1999!

When we joined the project, if the team wanted to [smoke
test](https://en.wikipedia.org/wiki/Smoke_testing_(software)#Origins) a design
they had to write C implementations of the key components involved. Doing so was
time-consuming and error prone, because C is fairly low level and is happy to
compile obvious mistakes. So Galois decided to see if using a higher-level
language could help speed up the prototyping process, by porting GHC to run on
Xen.  The initial revision of this work was by [Andrew
Tolmach](http://web.cecs.pdx.edu/~apt/), based on his prior work with
[House](http://web.cecs.pdx.edu/~apt/icfp05.pdf), which was based on a previous
system called hOp[^hop]. I joined Galois as he was finishing up his
work[^adamhs], and started tweaking Andrew's base port more towards our purpose
of building example microkernel-based systems. This became the HaLVM. I gave my
first talk on the subject in Cambridge in the Fall of 2006[^talkone].

[^hop]: I can't find a citation for hOp, but it was one of the first (if not the
    first) attempts to get Haskell running on bare metal.

[^adamhs]: My first Haskell program was a demo for this pre-HaLVM HaLVM, which
    probably explains something about my Haskell.

[^talkone]: "Hey, that sounds like [something one of our crazy dudes is doing
    here](https://mirage.io). Where'd [Anil](http://anil.recoil.org/) go?"

I include this early history partially to provide a wider view of the HaLVM over
time, but also to help the reader understand some critical requirements, and the
choices we made based on those requirements. As I recall, the critical
requirements worth mentioning in this document[^reqs] were:

 1. It had to run on Xen, on 32-bit systems, and provide direct access to the
    hardware.
 1. It had to allow the creation of many parallel, interacting components.
 1. It had to be "reasonably efficient"; we didn't need to use this system to
    generate benchmarks, but we did want to be able to draw reasonable
    conclusions regarding the efficiency of various designs.
 1. We needed to show our clients that this effort was valuable as soon as
    possible.

[^reqs]: I know, weasel words. There were three other requirements I'm not
    mentioning -- one regarding assurance road maps, another involving
    introspection, and another regarding network access during builds -- but they
    turn out to be superfluous for the purposes of this story.

These design requirements lead to many of the choices you see in early HaLVM
designs. For example, the emphasis on libraries and systems that use inter-VM
communication (IVC) streams, and the availability of as many low-level details
as possible.  However, there were two decisions we made at that time that we
will want to revisit later:

 1. *Get to Haskell as soon as possible.* Our theory was that by using Haskell,
    we could avoid huge classes of bugs, and thus get to working, evaluatable
    prototypes faster than if we used C. If this was true in general, it should
    also be true for us in building the HaLVM. Thus, we tried to get out of C
    as fast as possible, and build everything else we wanted in Haskell.
 1. *Compatibility is for the birds.* The HaLVM was an internal tool, likely
    only to be used for our purposes, and needed to be ready to go quickly.
    Thus, compatibility with a wide variety of libraries and other systems was
    unimportant.

HaLVM 1.0
---------

We released HaLVM 1.0 in 2010[^goddamndon]. This release was slightly tidied
from our internal version. In particular, we cleaned up some of the interfaces
and included some extra bits and pieces we thought people might need. After the
release, we started getting exactly the questions you think people would ask:
What libraries did we support? Could people run this on EC2?

[^goddamndon]: Admittedly, mostly due to a ton of nagging on the part of [Don
    Stewart](https://donsbot.wordpress.com/) and [Thomas
    DuBuisson](https://tommd.github.io/).

At which point we had a revelation: maybe we could use this for other things!
If you'd like to hear a little more about this phase of the work, I refer you to
[my XenSummit 2012
slides](http://www.slideshare.net/xen_com_mgr/the-halvm-a-simple-platform-for-simple-platforms).
These slides represent our early thinking that the HaLVM might eventually become
something more than just an operating system design tool. However, at this point
in time, our main use and goal for the HaLVM, internally, was still as a tool to
explore operating system component design. Beyond the core design reasons
discussed previously, we also used it to implement an IPsec layer that operated
on Xen[^6], for example..

[^6]: Sadly, this work is still closed source. Sorry!

HaLVM 2.0
---------

Eventually, though, we returned to the idea of compatibility, and using the
HaLVM for a wider variety of systems.[^buildcrazy]

[^buildcrazy]: Also, to the idea of making it easier to update alongside GHC,
    and to the idea of making the build process cleaner. Again. In the end, we
    only sort of managed the first.

There were three reasons that the HaLVM v1.0 might be incompatible with existing
Haskell libraries:

  1. Our `base` and other libraries had been modified to remove any notion of a
     disk, network, or other IO subsystem not available on bare metal.
  1. Our C library was extremely impoverished, so even Haskell libraries that
     used reasonable C faced an uphill battle in getting their unikernels to
     link.
  1. Unikernels are just a different environment than normal programs, and some
     libraries (GTK libraries, for example, or tight bindings to Linux services)
     just do not make sense.

In the HaLVM 2.0, we decided to try to tackle the first one by ensuring that our
base libraries included all the same functions and variables as the original GHC
base libraries. To do so, we reverted a previous decision. In previous versions,
if your program used a function that wasn't supported by the HaLVM, you would
find the functions in question simply missing, and the program would fail to
compile or link. In the new HaLVM 2.0, your program would link, and you would
get a runtime exception if you did something wrong. This change has always
struck me as less Haskell-y, but it certainly has had some great upsides.[^todo]

[^todo]: Perhaps someday we should add deprecation warnings (or some other
    warnings) to these functions and modules, so that authors are aware they are
    invoking functions that are guaranteed to raise errors. On the other hand,
    this would require even further modification of the `base` library, which
    may make the eventual goal of upstream support even more difficult.

The resulting system allowed us to compile any pure Haskell library available on
[Hackage](https://hackage.haskell.org), especially once we got Template Haskell
support working again. Plus a little bit of other work, and we had a fairly
flexible system. One invocation of `halvm-cabal` and you could download a
program from Hackage and build a complete HaLVM unikernel. Also, we had made a
number of other changes: turned our patches into a `git` branch on
[GitHub](https://github.com/GaloisInc/HaLVM), rebuilt our [network
stack](https://github.com/GaloisInc/HaNS) into something more complete, and added
SMP support[^smp].

[^smp]: Ever since I started talking about the HaLVM, I've been asked about SMP
    support, so I broke down and did it. So if you use the threaded runtime, and
    tell Xen to provide multiple virtual CPUs, it will try to use them. I'm not
    sure why people wanted this so much. It's always seemed like a better idea
    to write a bunch of unikernels that work together, but there you go. As it
    turns out, there's been a longstanding bug in this implementation, so I
    don't recommend using it.

We released the HaLVM 2.0 on [October 18th,
2013](http://community.galois.com/pipermail/halvm-devel/2013-October/000041.html),
and began using it on some of our internal projects, including
[CyberChaff](https://galois.com/project/cyberchaff/).

So What's Wrong With HaLVM 2.x? {#wrongwith2}
===============================

The HaLVM, in its current incarnation, has served us very well. As long as you
are writing new code, it is very easy to use. It has proven stable for a variety
of uses, and can be used to integrate with a wide variety of services and
microservices. As I mentioned in a [recent
talk](http://www.infoq.com/presentations/tor-haskell), writing unikernels is
even, at this point, kind of boring. Just keep a few design principles in mind
--- building and testing the program as a normal Haskell process until you're
ready --- and slip it into a unikernel as a final step.

But, in my opinion, there are four major problems with our 2.0 work:

  1. We still have all our old C incompatibilities. Building a Haskell
     library that includes C, or links with C, can be a challenge. Furthermore,
     the part of C that we do include is starting to show some cracks. We keep
     running into small bugs or incompatibilities in that library that cause
     hard-to-diagnose faults later on. Keeping up with the bugs and missings
     parts is an endless black hole.

  1. Our dependency on paravirtualized Xen is becoming a problem.
     Paravirtualized Xen will be around for awhile, but it is clear that the
     future of Xen is headed in a different direction:
     [PVH](http://wiki.xen.org/wiki/Xen_Project_Software_Overview#PVH) or one
     of its derivatives. In addition, while I'm a big fan of Xen for a variety
     of reasons, there are clear business cases to support other hypervisors
     as well.

  1. Widening our support for `libc` is one thing, but unfortunately it is
     not sufficient to meet our compatibility goals. Remember, all of our
     drivers, and thus all of our network and file system code, are written in
     Haskell. Thus, if the underlying C library uses a standard POSIX call
     (`bind`, for example), you will not get the behavior you expect. If we
     want to extend compatibility as much as we can, we must find a way to
     support libraries that expect network and file system access.

  1. It is very simple to talk about "the HaLVM" as if it is a single (albeit
     complicated) software system. The truth, however, is that like most complex
     software, "the HaLVM" is actually the conglomeration of a whole host of
     subprojects: a math library, a `libc` implementation, a port of GHC to a
     weird environment, a depressingly complicated build system, a set
     of device drivers, a few higher level libraries, and a series of scripts
     that make everything (more or less, sometimes) work together. Of course,
     that doesn't include the standard quality tasks regarding stability,
     testing, packaging, support, etc.

I have mentioned the following factoid to several people in person, but it bears
repeating: since the original funding for HaLVM development dropped off around
2009/2010, the HaLVM receives somewhere around 8 person-weeks of effort in a
good year. In bad years, that number drops to 2 or 3. If you do the math,
comparing this amount of time against the number of subprojects and their
support burden, you will notice an obvious problem.

Thus, the final goal for HaLVM 3.0 is to find other projects in the open source
world that are solving some of the same problems as the HaLVM, and leverage
their work whenever possible. If they don't solve the exact problem we have,
then finding something close may do: anything to shift some of the cost of
maintenance off to people who really want to solve the underlying problem(s), so
we can get more bang for our maintenance buck. Note, just to be clear, we want
to be good citizens of the open source world, and provide these projects with
feedback, patches, and other Good Stuff. We simply want to move towards using
infrastructure in which we are one of a group of contributors, rather than being
the only contributor.

HaLVM 3.0 {#halvm3}
===================

So let us get down to brass tacks: what are we doing for HaLVM 3.0? For this
update of the HaLVM, we are planning the following major changes:

  * Shifting the underlying C code for the HaLVM -- the GHC runtime adaptation
    layer as well as the `libc` and `libm` implementations -- to someone else's
    code.
  * Providing an abstract model of common subsystems, allowing developers to
    easily retarget their unikernels to different platforms and/or shift where
    the Haskell / C boundary lies.
  * Update the tooling, as necessary, to support multiple target platforms.

We will also, as usual, update to the latest stable version of GHC (probably
8.0.1).

Shifting to Other Platforms
---------------------------

Our major goal for this item is to shift away from supporting our own
hand-written assembly, `libc`, and glue code to someone else's work. At the
moment, I foresee two possible routes, both of which are probably worth
exploring:

  1. Shift to the [rumpkernel](http://rumpkernel.org). This would provide us a
     NetBSD-like base upon which to build the rest of the software, and would
     include both the underlying boot and driver code as well as a
     fully-functional `libc` implementation.
  1. Shift to a combination of [Solo5](https://github.com/djwillia/solo5) and
     either [musl](https://www.musl-libc.org/) or
     [uclibc-ng](http://www.uclibc-ng.org/), with stack extensions (probably
     [lwIP](https://savannah.nongnu.org/projects/lwip/) and some flexible file
     system) as required.

There are advantages and disadvantages to each approach. The rumpkernel is
nicely all-in-one, and the integration between the various pieces we would need
already exists. The downside is that it's all-in-one, and splitting apart some
pieces we might not need would be hard. The Solo5 route might make these splits
more simple, but would probably require us to write and maintain a glue code
library to support running the `libc` we chose in a unikernel environment.

### If We Choose The Rumpkernel ###

If we choose the rumpkernel, our first major change will be shifting the
lowest-level C libraries to use elements from the [rumpkernel
project](http://rumpkernel.org/). Our two main goals for this work are to:

  1. Inherit a fairly complete, extremely stable set of base C libraries from
     the NetBSD project, so we can stop maintaining our own.
  1. Leverage their support for both PV and bare-metal environments, so that we
     can stop maintaining our own low-level memory management and bootstrapping
     code.

Note, however, that I am being somewhat careful to not call this "porting to the
rump kernel," as I'm not sure that would be accurate. The rump kernel contains
quite a lot of code, much of which we do not want to use. It also has a project
roadmap that may diverge from the HaLVM in the future. For the moment, however,
the rump kernel solves many problems that the HaLVM 2.0 has exposed to us: the
brittleness of some of our underlying C code, the inflexibility of our
infrastructure, etc. So, as much as possible, we would like to take advantage of
all the work Antti and the rumpkernel team have put into it.

Early experiments in the `halvm3` branch suggest that this combination may be
fruitful. Using the rumpkernel as a base allows for a dramatic simplification in
the difference between the [HaLVM GHC
branch](https://github.com/GaloisInc/halvm-ghc/tree/halvm3) and stock GHC, and
early test virtual machines run acceptably. One possible concern we noticed in
this early testing was the size of the produced binaries. By playing around with
the libraries included in the `rumpkernel` build process, I was able to get a
"Hello, World" HaLVM down to 10MB unstripped, or 4MB stripped. 4MB is larger
than the current HaLVM's "Hello, World" (which is 2MB), but is close enough to
suggest that this transition path is reasonable.

Obviously, since I am capable of providing binary sizes, I have managed to get
the HaLVM running on top of rump. However, there remain some open issues with
this path:

  1. It is not clear how to split a rumpkernel installation into appropriate
     constituent sets of library. For example, how can we associate the
     rumpkernel file system libraries with a Haskell library that integrates
     with them? How can we be sure that libraries are loaded only when needed?
  1. At present, you specify the low-level platform for the rumpkernel tools at
     build time. This approach clashes with our eventual goal to offer
     retargetable HaLVMs. Will we need to build all possible variants of the
     rumpkernel to make this work?
  1. Is NetBSD's `libc` good enough? I'd like to think so, but `glibc` is fairly
     pervasive. Are we going to find out that we're running into problems by
     using a lesser-known `libc`?

Some of these questions we will only find the answers to by building the new
HaLVM and finding out what problems arise. Others we may be able to test early,
and find solutions to if we run into problems. As an example, we are actively
determining the extent to which we can re-use existing rumpkernel build
infrastructure, and to what extent we may suggest modifications.

Finally, switching to the rumpkernel changes the architecture of the HaLVM build
system in some interesting ways. Because of how it is implemented and
distributed, we would simply build GHC using the rump configuration tools, and
then use other rump tools (along with some specialized shell scripts) to
generate the final binaries. This method is a bit different than our current
methodology, in which we insert a bunch of code into the GHC build process to
tie some of the knots for us. This change is not a bad thing -- in fact, it
might be a good thing, as it simplifies our changes to GHC -- but it will
require a fresh build architecture, and thus may require us to repeat the effort
of generating appropriate packages for various platforms.

### If We Choose Solo5 ###

In contrast, a port to Solo5 would be more like a "simple"[^hahaha] replacement
of [our existing Xen
port](https://github.com/GaloisInc/halvm-ghc/tree/halvm/rts/xen) with Solo5, and
[our existing `libc` implementation](https://github.com/GaloisInc/minlibc) with
whichever version we choose. Then all we have to do is get everything to build
and we're home free. Simple.[^hahaha]

[^hahaha]: HAHAHAHAHAHAHAHAHA I'M SO FUNNY.

Solo5 directly targets 64-bit x86, and is particularly targeted at KVM/QEMU.
However, the `virtio` substrate it uses is widely supported across a wealth of
platforms, so it should transparently port to a number of useful architectures.
Not, however, as many architectures as rump is targeting, though, and without
the huge history of the NetBSD project behind it. Which is good and bad.

After the initial step of getting Solo5 to work as a base layer for the HaLVM
was done, the next step would be integrating a larger `libc` along for the ride.
There are, fortunately, several options to choose from. Clearly, the deciding
factors will be compatibility with other `libc`s (and `glibc` in particular), as
well as ease of retargeting for alterate platforms. A likely problem will be
supporting `pthreads` in an interesting way; this may require an update to
Solo5.

### Evaluating The Final Path ###

If it's not clear, while I was initially leaning very strongly towards the
rumpkernel, this decision is still very much up in the air. It has been clear to
me that the rumpkernel provides a huge number of advantages to a system like the
HaLVM, but does so at some cost in complexity. A system like Solo5 may
dramatically reduce that complexity, but at the cost of more work on our end.
Just to confuse the matter, it may also be interesting to consider a HaLVM that
includes both.

As we move forward, though, whichever solution we choose will be evaluated base
on the following criteria:

  1. The extent to which if offloads work from the HaLVM maintainer team.
  1. The extend to which it supports a wide variety of Haskell libraries.
  1. The size and performance of the resulting binaries.

Initially, it looks like the rumpkernel may provide easier, wider library
support, while Solo5 is designed for performance. Thus, we are investigating how
significant each of these "leads" is, and how much work we need to impart to get
these benefits.

### Why not switch to *X*? ###

There are a couple other platforms that we could switch to, rather than the
rumpkernel. To my mind the major contenders would be [OS/V](http://osv.io) and
[Mini-OS](http://wiki.xenproject.org/wiki/Mini-OS).   Just to be clear, I like
both of these projects, I just don't think they're the right fit at the moment.
Why?

OS/V is a very interesting project designed to allow the author of any POSIX
program meeting a couple limited guarantees (for example, it can't invoke
`fork()`) to run on bare metal. It is very cool, and can run on a variety of
interesting architectures. My concern with OS/V is the ease to which it can be
cut down based on the functionality in the program, compared to the rumpkernel's
ability to do the same thing. The rump tools already provide ways to select
subsets of libraries to include in your unikernel; OS/V's tools do not appear to
support the same capabilities. In addition, OS/V is 64-bit only, and it is nice
to be able to support 32-bit hosts, as well.

Mini-OS is also a very cool projects, and is very similar in its goals and
methods to Solo5. They also share the same disadvantage: the need to integrate a
third-party `libc`. Mini-OS is Xen-only, though, and thus Solo5 offers a more
interesting pathway to a wider variety of platforms.


`HaLVMInterfaces` and The New Library Regime {#interfaces}
--------------------------------------------

The next major change to the HaLVM will be restructuring the libraries in such a
way that application and library authors can write implementations over abstract
interfaces, leaving the final choice of implementation or platform to the last
moment. The core of this system is the
[HaLVMInterfaces](https://github.com/GaloisInc/HaLVM/tree/halvm3/src/HaLVMInterfaces)
library, currently under development, and eventual deep integration with the
forthcoming [Backpack extensions to Haskell](http://plv.mpi-sws.org/backpack/).

The plan for `HaLVMInterfaces` is that it contain a set of high-level
descriptions of low-level systems: devices, network stacks, file systems, etc.
Library and unikernel authors can then write their implementations over these
interfaces, which can be implemented in multiple ways. For example, a library
could be built on top of a "network card driver", which could be instantiated
based on a real device driver, a driver for a Xen paravirtualized device, or a
`virtio` driver.

(This technique is very similar to any number of features of any number of other
languages. Perhaps the most obvious is its similarity to ML modules, which allow
the same nice separation of interface from implementation. You could also
imagine that I'm trying to generate a weaker version of
[units](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.14.4663), with
Cabal serving as the linking system. Java and other OO programmers will also see
the obvious analogy to Interfaces and Classes, translated to the module and
library level.)

In addition, we also hope to use this same mechanism to solve some problems in
linking with foreign C libraries, by allowing users to choose the point at which
the HaLVM transitions from C to Haskell. Presently, this transition happens very
early in the boot process, and all of the drivers and system stacks are
implemented in Haskell. This choice provides some nice properties, but means
that C libraries expecting disk or network support at the C level are left out
to dry. In the HaLVM 3.0, we plan to allow this transition to happen at various
points, based on the needs of the system. The following table lists the targets
we currently envision, along with information about where the C / Haskell
transition occurs:

Platform Name  | Bus Level | Driver Level | Stack Level | Application
---------------|-----------|--------------|-------------|------------
xen-pv         | Haskell   | Haskell      | Haskell     | Haskell
xen-pv-cbus    | C         | Haskell      | Haskell     | Haskell
xen-pv-cdrive  | C         | C            | Haskell     | Haskell
xen-pv-clib    | C         | C            | C           | Haskell
xen-pvh        | Haskell   | Haskell      | Haskell     | Haskell
xen-pvh-cbus   | C         | Haskell      | Haskell     | Haskell
xen-pvh-cdrive | C         | C            | Haskell     | Haskell
xen-pvh-clib   | C         | C            | C           | Haskell
xen-hvm        | C         | C            | Haskell     | Haskell
xen-hvm-clib   | C         | C            | C           | Haskell
kvm            | Haskell   | Haskell      | Haskell     | Haskell
kvm-cbus       | C         | Haskell      | Haskell     | Haskell
kvm-cdrive     | C         | C            | Haskell     | Haskell
kvm-clib       | C         | C            | C           | Haskell

This level of variation should provide unikernel authors with the flexibility to
build their unikernels in the way that makes sense for them. In particular, our
hope is that people will be able to use any of the Haskell web frameworks, out
of the box, with the `*-clib` targets. These variants will provide the necessary
functions at the C level that these libraries currently need. Over time, these
initial unikernel implementations may shift away from these stock frameworks,
and shift to frameworks that support a larger proportion of Haskell. Again, we
firmly believe that there is a great deal of value to getting to Haskell as soon
as possible: the goal of this work is just to provide an on-ramp for people
constrained to working with C.

In addition, at some point --- probably after 3.0 --- I would like to see the
HaLVM expand to support a wider variety of platforms, including classic
operating systems. Specifically, I'd like to see the same tool chains used to
compile a HaLVM to Xen, to OS/X, to Mach, to SEL4, or to bare metal, with just a
change in flags guiding the compilation.

Finally, in some sense, this change runs counter to our desire to lower our
maintenance burden in HaLVM 3.0 , as we are now supporting a lot more variants
of "the same code." To this, I counter with the other goal of the HaLVM, which
is to allow for a wider variety of target systems. Any multi-target system will
require some of this work. That being said, we are clearly adding a lot of
duplication; we just hope that this duplication is easy to test, and is more
than made up for by the other changes we are making.

Updating the Tooling
--------------------

In my experience, the best way to develop a HaLVM-based application is to build
it for a more normal platform first, and then port it to the HaLVM when you're
sure that it is working. Sometimes, however, this is easier said than done. The
HaLVM supports only a certain subset of Haskell libraries, and some of these
libraries can only be used in limited ways. Thus "porting it to the HaLVM" can
require enough work that it is deceptively easy to introduce flaws in your
software in seemingly-innocuous steps.

The end goal for the HaLVM is to make the change between building a program for
your normal operating system and building a unikernel as transparent as
possible. Ideally, it should simply be an option to the compiler, a different
name for the build tool, and/or linking in a different library.

Clearly, the interfaces discussed in the previous section play a significant
role in this effort. However, I believe that there will need to be a similar
level of effort invested into updating the tooling around the HaLVM to support
easily switching between platforms. Providing this support is somewhat foreign
using existing tools, given the weak support GHC includes for cross-compiling.
Still, our goal for the HaLVM3 is the ability to quickly and easy transition
back and forth between developing for bare metal and developing for a
standardhost operating system.

Tactics {#tactics}
==================

Development of the HaLVM v3.0 has begun, at least in experimental form. We would
love to include more people in this process, and I am hoping that this section
can provide people new to the project with some guidance on how to get started.
Let me say two things, though, that I'm hoping won't be contradictory:

First, the HaLVM will not survive if it continues to be the part-time project of
a few engineers in and outside of Galois. We very much want to include new
people, at all skill levels and all levels of experience with Haskell. Remember,
the HaLVM was my first Haskell project! It can be yours, too! So, if you read
this and still aren't sure what to do next, or aren't sure what some process is:
please ask. Don't be intimidated.

That being said, please, please have patience with us. The HaLVM is not our full
time job. We have other projects that demand our attention, and frankly, paying
clients will always come first. So understand that we really want to answer your
question, but it may be a few days (or even a few weeks) before we can get to
it. Sorry! But we will answer. Especially if you nag us.

With that introduction, let us dive into an unsorted bundle of tactical issues.

Forums and Conduct
------------------

Shortly after I post this document to the global interwebs, I will post a link
to it on [devel.unikernel.org](http://devel.unikernel.org). My hope is that this
post will become the main forum for high-level discussions about HaLVM v3.0
goals, release criteria, etc. We may use other posts there for announcements,
deeper discussions on other topics, etc., so watch that space.

In addition to this space, we will be using [GitHub
Issues](https://github.com/GaloisInc/HaLVM/issues) as the primary mechanism for
discussing specific bugs or features. We have a shiny new green "v3.0" tag now
available for [marking items as 3.0-relevant or
not](https://github.com/GaloisInc/HaLVM/labels/v3.0). Please use it! We may also
create a v3.0 milestone marker, as well, once we figure out if we like
milestones.

Since I am potentially creating forums for discussion on the Internet, let me
mention two things.

First, I hold a deep appreciation for [this XKCD comic](https://xkcd.com/169/):

![](http://imgs.xkcd.com/comics/words_that_end_in_gry.png)

Second, I appreciate the simplicity and clarity of [the FreeBSD Code of
Conduct](https://www.freebsd.org/internal/code-of-conduct.html), and will be
adopting it for the foreseeable future. If you see a violation of this code of
conduct, please direct it to me, [Adam Wick](mailto:awick@uhsure.com). In the
future, should the HaLVM grow beyond a few contributors, I will make sure that
there is a leadership committee tasked with reviewing the Code of Conduct and
creating a task force to deal with any violations. For now, though, let's keep
it simple.

Branching and Merging
---------------------

All development of the HaLVM occurs on the [GitHub project
page](https://github.com/GaloisInc/HaLVM). In discussions with people, I
occasionally run across the belief that Galois has a secret, internal version
that we occasionally push out. This is, in fact, not true. We use the public
version, just like everyone else.

If you would like to send patches, feature enhancements, documentation
enhancements, etc. to me, please do so in the form of GitHub pull requests.
These are much, much easier for me to deal with than email or whatever other
mechanism you choose. I would also highly recommend a read of GitHub's quick
guide to [writing a good pull
request](https://github.com/blog/1943-how-to-write-the-perfect-pull-request).
There is not much more I can expand on this subject.

The only other thing I will say is that I tend to prefer clean Git histories.
You'll notice, in this and other projects, a tendency of mine to use feature
branches and the [squash
merges](https://github.com/blog/2141-squash-your-commits).  I do so because I
like having the minute history of a branch available to me if I really want to
go find it, but don't want to see it all the time when I read the project log.

I won't reject a merge because it's got a few "wibble"s or other tiny
corrections or changes in the history... but please keep in mind that I'd love
it if you wouldn't submit them.

Diving In
---------

The first experiments that will become the HaLVM v3.0 are operating in the
[standard HaLVM tree](https://github.com/GaloisInc/HaLVM) in the
[halvm3](https://github.com/GaloisInc/HaLVM/tree/halvm3) branch.

Because I wasn't thinking at the time, this branch contains a bunch of small
changes and experiments, almost completely contradicting my previous remarks.
Please do as I say, not as I do, and I will try to follow my own rules, as well.

Shortly, the existing `halvm3` branch will become the "pre-release master"
branch for the v3.0 project. To that end, most development should happen in
separate feature branches, and (squash) merged back into `halvm3`. As an
example, I will probably continue the rumpkernel experiments in a forthcoming
`rumpkernel` branch. If someone wants to play around with the HaLVM on Solo5,
let me suggest a super sneaky branch name like `solo5`.

Similarly, I'm going to start an additional branch that focuses on the
`HaLVMInterfaces` work, probably with a similarly-inventive name.

For those that want to help out with the HaLVM v3.0, let me suggest FIXME major
efforts that I can imagine would be good places to dive in, and then expand on
them for the rest of this section:

  1. Write a HaLVM unikernel for some project of yours. Note that this does
     not need to be a v3.0 unikernel! Feel free to use the v2.0 trees. (In
     fact, I'd encourage you to do so.) Why do I list this as a v3.0 task?
     Because the more mileage we can get out of v2, the better decisions we
     can make about what to keep or change in v3.

  1. Pick [an open issue](https://github.com/GaloisInc/HaLVM/issues), add a
     comment saying you're looking into it, and go! If you're specifically
     interested in HaLVM v3 issues, they should be tagged as such.

     I will do my best to keep a continuous supply of small- and mid-sized
     projects in that pool for people to work on. However, another way to
     get started would be to file an extension you'd like to see. (Picking
     it up as a developer is optional, of course!)

  1. Consider experiments with the HaLVM / rumpkernel or HaLVM / Solo5 options.
     At this point, I have very mixed feelings about both approaches, and
     can always use more data. I've mostly been pushing down the rumpkernel
     line so far, because "all in one" is very tempting. That being said,
     there's a lot of (deserved) excitement about Solo5. Is anyone interested
     in starting to investigate that line of development?

     Even more broadly: how would you choose between the two? What are your
     critical axes for comparison? Binary size? Performance of key functions?
     Number of supported platforms? We should gather this info and put it
     together; [here's a wiki
page](https://github.com/GaloisInc/HaLVM/wiki/HaLVM-v3.0-Platform-Thoughts) that
     we can use to start condensing our thoughts on the matter.

  1. Consider the
     [HaLVMInterfaces](https://github.com/GaloisInc/HaLVM/tree/halvm3/src/HaLVMInterfaces)
     library. What other core devices might we want to export? Indeed, at the
     moment, we're only looking at devices, which is short-sighted. We should
     also consider exporting interfaces for network stacks, disks, etc. What
     are we missing? What do these devices look like?

     If there's a device or subsystem you think is missing, then let's get it
     in there. What do you think its interface should be? Send in a pull
     request with your new device or service and its interface, and let's
     start the ball rolling.

  1. Let's flip to the other side of `HaLVMInterfaces`: the implementations.
     What does it take to build some of the possible connections I laid out
     in my [high-level goals](#interfaces)? Let's start smoke testing the
     interfaces we're designing by seeing how hard it is to implement these
     interfaces for various platforms.

Final Thoughts {#ack}
=====================

The work I have described in this document is significant. I am personally
committed to pushing the HaLVM further, I know several other historic
contributors are equally interested in helping out, and we would love any help
that you can provide. With that said, I expect that the development of HaLVM v3
will consume at least the rest of 2016, and perhaps even into 2017.

While we work on it, we will continue to work on improving and supporting the
v2.0 branch. In particular, we are pushing very hard on decreasing the learning
curve (and investment costs) for developing HaLVMs, through the creation of
pre-built packages, Vagrant and Ansible tooling, and Docker integration. We will
also be looking at tooling to simplify the process of developing and deploying
HaLVMs into EC2 and other cloud platforms. So keep watching for news on those
fronts.

I would also like to take a moment to thank all the wonderful people who have
contributed to the HaLVM or one of its related projects over the years: Andy
Adams-Moran, Magnus Carlsson, Nathan Collins, Iavor Diatchki, Tom DuBuisson, Tim
Dysinger, Trevor Elliott, Michael Hueschen, Rebekah Leslie-Hurd, Wei Liu, Eric
Mertens, Tim Smeets, Joel Stanley, Don Stewart, Mark Wotton, Huang Yi, and Zhen
Zhang.

Finally, please direct any questions, comments, or thoughts regarding the HaLVM,
including the v3.0 effort, to [the unikernel discussion
board](http://devel.unikernel.org), tagged with the HaLVM. Alternatively, I can
be reached at [awick@uhsure.com](mailto:awick@uhsure.com), if you'd prefer to
rant at me in private.

If you've made it to the end, congratulations, and I thank you for your time and
attention. If there's anything I can do to make the HaLVM easier for you to use
or contribute to, please don't hesitate to [file an
issue](https://github.com/GaloisInc/HaLVM/issues/new) or simply write to either
of the places above.
