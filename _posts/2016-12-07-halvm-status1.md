---
layout: post
title: The HaLVM Status Report, Issue 1
date: 2016-12-07 16:00:00 -0700
author: awick
categories: status
---

One of the things we're trying to get better about, here at HaLVM headquarters,
is communicating with the wider HaLVM and unikernel communities. In particular,
during those periods when we're slowing building out upgrades, documents, or
other things for the HaLVM, we tend to go silent. I've heard rumors that this
silence has led people to believe that there's some deeply complicated internal
development and release process within Galois; nothing could be further from the
truth. The truth is, basically, that I'm lazy, distracted, and not particularly
good with markdown. [Hanlon's
Razor](https://en.wikipedia.org/wiki/Hanlon's_razor) wins again.

With that introduction out of the way, let's chat about the three major
initiatives going on at HaLVM Headquarters, and how interested people might jump
in:

  1. [Initiative #1](#halvmdotorg): The new [halvm.org](http://halvm.org). As
     part of our continuing outreach and documentation effort, we're creating
     [halvm.org](http://halvm.org): your one stop shop for HaLVM news and
     information. [Halvm.org](http://halvm.org) will, of course, be hosted on
     the HaLVM and will likely run on Amazon's cloud.
  1. [Initiative #2](#v2improve): HaLVM v2 improvements. While we're working on
     HaLVM v3, we want to be responsive to the needs of existing HaLVM users, or
     new users that are excited about the project.
  1. [Initiative #3](#newhotness): HaLVM v3. The next generation of HaLVM
     awesomeness is in progress.

Please jump to the one you're interested in, or stick around for all three.

# The New Halvm.org {#halvmdotorg}

Presently, [halvm.org](http://halvm.org) simply points to the [HaLVM GitHub
page](http://github.com/GaloisInc/HaLVM). This choice was the simplistic one,
and I'll admit that I'd hoped that the wiki therein would become a popular
place. However, there's some serious problems with a project's GitHub page as a
news distribution site. So we need a new one.

At the same time, there's a bunch of stuff that we'd like to link to, and we'd
love to have a place for people to post HaLVM tutorials, news, insights, and war
stories. The new [halvm.org](http://halvm.org) is going to be that place.

For those that want to know what's under the hood, it's going to be a
[Jekyll](https://jekyllrb.com/)-based website running on the HaLVM on EC2.
Ideally we'll set it up so that the website is a public repo, so we can accept
contributions to the site via GitHub pull requests ... but no promises. We're
still working on the mechanics, at this point.

Still, look for an announcement, hopefully before the new year, about the new
HaLVM.org.

# Improving HaLVM v2 {#v2improve}

While we are hard at work trying to work on v3, we're still supporting v2
requests as we do. At the very least, we're using the HaLVM as the base of [the
CyberChaff network defense system](http://cyberchaff.com) and
[halvm.org](http://halvm.org), so it's important that we keep working to improve
the capabilities and reliability of the platform.

## HACKING.md

Thanks to Zhen Zhang (GitHub's [izgzhen](https://github.com/izgzhen)) and Arnaud
Bailey (GitHub's [abailly](https://github.com/abailly)), we now have a nice
markdown file in the repo that provides an intro to hacking the HaLVM. This file
will hopefully replace the myriad, all slightly broken, files and wiki pages,
and it's built into the repo to boot! See the brand-new, shiny `HACKING.md` in
the mainline repo.

## EC2 Support

Depending on when you last read up about the HaLVM, let me make sure to
announce: the HaLVM runs great on EC2.

Currently, we strongly suggest building and experimenting with your HaLVM-based
product on a local machine, and only uploading it to EC2 once you're pretty sure
that things work as expected. This is for a couple reasons. First, debugging
systems running on EC2 is a bit more difficult than debugging something locally
on your machine. Second, EC2 is really cool infrastructure that runs pretty fast
once you get a VM uploaded, but it is not designed for super-speedy imports.
Thus, pushing something from your local computer to EC2 is a 5-20 minute
process.

By the way, that process is a little grungy if you do it by hand: you need to
build a hard disk image, get a few grub files right, get it uploaded and
imported, etc., etc. We suggest using
[ec2-unikernel](https://github.com/GaloisInc/ec2-unikernel), instead.

## The Memory Dragon Loses a Couple More Heads

Since the birth of the HaLVM, there has been this annoying problem with regard
to memory. The simplest way to explain it is thusly:

There are three users of memory in the HaLVM: the binary blob that is your
program, the garbage-collected Haskell heap, and the miscellaneous
manually-allocated bits and pieces that happen while the system operates. When
you allocate a 64MB HaLVM, for example, you split that 64MB between these few
things. (If I had to guess, it's probably split 3-8MB for the binary code that
runs your system, 2-4MB miscellaneous manually-allocated bits, and the rest GC'd
heap.

Here's the thing, though. GHC's memory management system, like most POSIX
programs, defaults to the assumption that it has an infinite amount of memory
available to the GC'd heap. This is bad in the case of the HaLVM, because we
have very limited memory. So we pass the runtime a flag at boot time that says
"GHC, you should stick within this amount of memory."

There are two problems with this.

First, annoyingly, GHC plays a little fast and loose with this number. It seems
to take it as a strong guideline rather than a law: it tries hard to stay under,
but seems to feel free to go over by a little.

Second, and more importantly, is the following question: what number should we
tell it? Unfortunately, this requires some psychic powers, as we need to predict
the maximum amount of "miscellaneous" memory that the system might use in the
future.

So we guess, and it works in many cases. But sometimes its wrong, and when it's
wrong things crash. If you're using the `halvm-xen` packages we distribute for
Fedora, and [have the proper logging flags
set](https://github.com/GaloisInc/HaLVM/wiki/Building-a-Development-Virtual-Machine#install-xen-and-the-halvm),
then you'll probably see a message like this:

```
WARNING: Out of memory. The GHC RTS is about to go nuts.
```

If you've ever wondered exactly how or why this happens, now you know.

With all that background in mind, one of the ways we've been trying to improve
our ability to do this prediction is by removing the amount of "miscellaneous"
memory used by the HaLVM during operation. The changes in HaLVM 2.2 are mostly
in regard to this, and move significant sources of manually-managed memory into
the GHC heap. So far this appears to increase the stability of the system.

Long term, our goal is to remove the manually-managed memory system entirely,
but until then we'll keep chipping away.

# HaLVM v3 {#newhotness}

Work on the HaLVM v3 continues. Mainline development is now in the
`wip-halvm3-base` branch, which will eventually be merged into the `halvm3`
branch once we're a bit more confident of our direction and have a full system
building at least "Hello, world!"

## The HaLVM v3 Design

"But wait!" I hear you say, "There's a direction?! [The last time you mentioned
HaLVM v3](http://uhsure.com/halvm3.html) you weren't sure which way you were
going to go!"

Well, now we do. And it's Option #2, with a caveat. But perhaps the best thing
is to show you a picture.

![]({{ site_url }}/assets/halvmarch.png)

Let's walk through this.

At the bottom, we have our target architectures. At the time of release, we'd
like to target three core architectures: paravirtualized Xen, KVM, and the raw
Linux kernel. In addition, we'd like to target other POSIX systems in general,
including macOS. But just in case you're questioning my math: the last is a nice
to have for release. I really want the first three.

For each of these systems, we need a base implementation that supports critical
tasks such as booting and raw memory management.
[Mini-OS](https://wiki.xenproject.org/wiki/Mini-OS) can serve this purpose for
paravirtualized Xen, and
[Solo5](https://developer.ibm.com/open/solo5-unikernel/) can serve this purpose
for KVM. We'll design similar -- probably extremely basic, possibly nonexistent
-- versions of these systems for the Linux kernel and POSIX.

Now, let's skip the "low-level system support interfaces" layer for the moment,
and jump to `musl`. [`Musl`](https://www.musl-libc.org/) is a super cool
lightweight implementation of `libc`, and looks to be extremely compatible with
`glibc` and other systems. We're going to use `musl` as the basis for compiling
the GHC runtime as well as any other C libraries that run within your unikernel.
This will allow us to drastically cut down the number of changes need to GHC to
turn it into the HaLVM, as well as allow HaLVM users to link in a much wider
variety of C and Haskll libraries. (Including, very critically, Haskell's
`network` library.)

In order for `musl` to do its thing, however, it requires a connection to "the
host operating system." In particular, `musl` expects a Linux kernel around to
shunt operating system requests to. Which is a little tricky, as there isn't one
in this case. So what are we going to do?

Well, if you look in the `wip-halvm3-base` branch, you'll see: we're going to
trick `musl` out, by rerouting system calls to our own set of core functions.
These will be implemented by the aforementioned "low-level system support
interfaces". Depending on the particular system call in question, these will
either:

  * Return something nice and bland, like `ENOSYS` or `EINVAL`.
  * Kick the request over the underlying support system (solo5, etc.)
  * See if there's a handler for this call installed at the Haskell level, and
    ...
     * if so, run it
     * if not, return a boring thing, as above

The last bit is the really interesting bit. We're going to set this up so that
applications and libraries can reroute these core OS calls back into Haskell.
The first and most obvious target for this is our Haskell network stack,
[`hans`](https://hackage.haskell.org/package/hans). With this capability, you'll
be able to use `hans` as your network stack *and* use `network` for your server,
by wiring the C socket calls to invoke `hans` as appropriate. Have your Haskell
system service cake and eat your legacy code, too!

## The Next Steps

This is all in progress. Watch the status of the `wip-halvm3-base` branch for
updates. What are our next steps?

  * We have `musl` building against a `halvm` target, which redirects system
    calls into the appropriately-named function prefixed with `system_`. For
    example, instead of using the appropriate x86-64 trap to invoke the system
    call `fcntl`, we instead call the function `system_fcntl()`. So that's
    great! But now we need to implement these system calls, and there's a bit
    over 200 of them.

    Most of them are very similar, which should be nice. And the ones that all
    talk to the same subsystem (like all the networking calls, or all the file
    calls) should have a very similar interface, in which they see if there's a
    Haskell handler and try to jump to it if possible.

    So the next task, which I've taken some stabs at but haven't been completely
    happy with, is to build these out. Preferably using C and Haskell macros to
    limit the number of flaws we introduce in this layer.

  * Skipping up a couple levels, we'd like people to be able to very easily mix
    and match different combinations of underlying systems and libraries. More
    concretely, we'd like people to easily switch not only between a HaLVM that
    uses Xen and KVM, but also between a HaLVM that uses Linux only for the
    network driver (with Hans as the network stack) and a HaLVM that uses Linux
    for its entire network infrastructure. To do so, it seems like it'd be nice
    to have a standard set of interfaces for common devices and operating system
    services.

    I started sketching some of these out in the `halvm3` branch, in the
    `src/HaLVMInterfaces` library. The idea is that HaLVMs could write to these
    interfaces in general, and then you could very easily switch between them.

    So another task might be to start thinking about this work, again, and how
    we can best implement interfaces for disks, network cards, file systems,
    network stacks, consoles, etc. Perhaps even more exotic devices, as well?
    USB? Sound? Frame buffer graphics?

# Conclusion

Well, that's where we are in HaLVM-land. As we can, we're pushing forward on as
many fronts as we can. The more the merrier! If you'd like to help out, jump in!
If you're not sure where, check out the GitHub issues. Or, alternatively, just
shoot us an email with the sorts of things you'd like to do, and we can try to
point you in the right direction.

Thanks for reading, and we'll look forward to posting Episode 2, with updates on
these initiatives and any new things we're working on!

