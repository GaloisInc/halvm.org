---
layout: post
title: StrangeLoop Workshop Materials
date:  2016-09-19
author: awick
categories: article
---

Just a quick note to announce that the materials we used for [our StrangeLoop
workshop](http://www.thestrangeloop.com/2016/unikernel-microservices-build-test-deploy-rejoice.html)
are now available for general consumption. In particular, we have a [virtual
machine](http://repos.halvm.org/strangeloop2016/unikernel-workshop.ova) all set
up and ready to go with the HaLVM, Docker, Mirage, and the rust tools. It should
come with all the hypervisor and networking tools ready to go. In addition, you
can also walk through the [slides I
presented](http://repos.halvm.org/strangeloop2016/Strange%20Loop%202016%20Workshop.pdf)
at the beginning of the workshop.

A couple quick notes, for dealing with the virtual machine:

  * When unpacking these machines **do not change the MAC addresses of the
    NICs.** We have the internal network configured based on those MAC
    addresses, so if you change them nothing will work too well.

  * Do not use this virtual machine in any context in which its security is
    important. There is a public website with the username (`user`) and password
    (`unikernel-workshop`) of a user with `sudo` access. Also, we disabled the
    firewall on the VM. So be careful.

  * We set up our testing VMs with two NICs, to make testing easier. The first
    NIC is a NATted connection to the larger Internet available from your host
    machine, and is used because developing without access to the Internet is
    torturous. The second is a "host only" NIC used for testing unikernels. This
    NIC is connected in the VM to a bridge (`xenbr0`) that your unikernels will
    attach to, and supports DHCP for said unikernels. You can then address these
    from inside the VM or from your host machine, as you need, but they will not
    be addressable from off the host.

