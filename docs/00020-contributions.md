# bpftrace

What drew me to bpftrace was an early post [@bpftrace-brendan-gregg] about it
by Brendan Gregg after it first got pulled into the iovisor org, from Alistair
Robinson's [@ajor] initial repository.

It was a pretty cool project! I had been playing around with bcc for a few
months, trying to use it to trace network issues we were experiencing in our
Kubernetes clusters, and see if eBPF could be used to help pin down other
problems as well, including tracing the containers inside our Kubernetes pods.

I saw right away that a lot of the bcc scripts had been re-implemented in
bpftrace scripts - cool! It was amazing to me to see that large amalgams of
python and inline C code could be simplified down to just a few lines (or in
some cases, getting most of the functionality from one single line!) of
bpftrace.

I saw from Brendan's post that [@bpftrace-brendan-gregg] bpftrace's syntax was
easily translatable from dtrace, they were nearly on par with one another. I
had heard of dtrace before, from some debugging my colleague Burke Libbey had
done in our development environments on Darwin (OS X). It baffled me that such
a powerful tool existed bundled into ever one of our laptops, but that there
was no such analagous tool for Linux.

I had been following kernel tracing since 2013, as I had been trying
(unsuccessfully) to implement Systemtap support into a side-project of mine,
and find other frameworks to work with kprobes and uprobes in production. The
overhead systemtap probes was a bit scary though, and it required loaded a new
Kernel module, which wouldn't work on Chromium OS derivatives for security
reasons, so wasn't viable for production use.

eBPF promised an intriguing new frontier - a way to access the kernel's uprobe
and kprobe API, **built-in** to the linux kernel, and on top of the existing
BPF infrastructure already employed for enabling tcpdump from kernel space.
Unlike Systemtap, eBPF would work without issue on Chromium OS derivatives, and
so was more viable for production usage.

## Contributions

When I come upon a missing feature in an open source project, I might be at
at first disappointed, even dejected, but sometimes I instead become  really,
really excited - it means that **I** could be the one to implement it, if I put
in the time to understand the problem and come up with a workable solution!

This is the best type of problem to solve - one you have yourself, as you get
to act as your own QA / customer, and benefit directly from the new feature.
To add to this, you can benefit from expert code reviews you wouldn't otherwise
have access to, and learn to collaborate with people all over the world.

## Gaining Context for first Contribution

The first problem I encountered was that while a number of the other bcc
scripts had been ported to bpftrace, there were none of the tcp tools available.

This was disappointing, as network tracing was one of the main use cases I had
for eBPF. From Brendan Gregg's 2017 Velocity talk [@velocity-brendan-gregg-2017]
, which I actually had the pleasure of attending in person, I learned that eBPF
was able to provide a lot of the value of tcpdump, without instrumenting `send`
or `receive`, but my being much more targetted on specific types of interesting
events (or tracepoints).

If I was really going to use bpftrace, it would need to have at least the same
utility for debugging network issues as the existing bcc tools did. So, I went
ahead and filed an issue [@bpftrace-issue-245] for this, then looked to what
would be needed in order to solve this myself. As a rule, I try to only submit
an issue if I have some idea for how **I** might implement it, or some
intention of being the one to own it and see it fixed.

I read through bpftrace's internals development docs, which gave an overview
of how bpftrace uses LLVM to generate eBPF. Basically, it revelead how bpftrace
uses the LLVM framework for code generatino, that ultimately become compiled
eBPF instructions that can be loaded into the Linux kernel. Cool!

There were even some examples that walked through how other functionality was
added to bpftrace, so I could use these as a practical reference point and a
"reverse-engineering" manual, to see how I could use the same concepts to
implement this for myself.

// FIXME - put some references to the internals docs here?

