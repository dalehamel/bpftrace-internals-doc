---
title: Contributing to bpftrace
author: Dale Hamel

header-includes: |
    \usepackage{fvextra}
    \DefineVerbatimEnvironment{Highlighting}{Verbatim}{breaklines,commandchars=\\\{\}}

---

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

# Contributions

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

## Adding ntop to bpftrace

In order to debug network traffic and implement tcp tracing scripts for
bpftrace, the main thing I knew was missing was the ability to print IP
addresses. I decided to tackle the existing issue for implementing ntop
[@bpftrace-issue-30]. So I set to work on this, which was my first introduction
to bpftrace, and got to work on [@bpftrace-pull-269].

### Lexer and Parser

bpftrace uses yacc to formally describe its syntax and the symbols of the
language, which is used to parse the bpftrace AST.

To add a new builtin function, we rely on the `call` type of expression:

```{.yacc include=src/bpftrace-269/src/parser.yy startLine=180 endLine=186}
```

Which we see will be the treatment for when an expression uses paretheses,
accepting a variable size argument list:
```{.yacc include=src/bpftrace-269/src/parser.yy startLine=220 endLine=222}
```

When the parser encounters a function call then, it actuall passes the identity
of the function (the identifier that preceeded the parentheses) as the first
argument, then the argument list (the third parser token) as the second
argument to a new `Call` AST node. As for the variable argument list, we can
see it is just an AST node for an `ExpressionList`:

```{.yacc include=src/bpftrace-269/src/parser.yy startLine=231 endLine=233}
```

So all this to say, as long as the caller follows these syntatic conventions,
a new Call node should be created:

```{.c include=src/bpftrace-269/src/ast/ast.h startLine=58 endLine=66}
```

This shows us that initializer lists are used to set the `func` and `vargs`
members of this class, so that these values can be accessed by the semantic
analyser to decide how to handle this.

### Semantic Analyser

As shown above, the parser doesn't actually have any idea if a function is
actually implemented. That is up to the semantic analyser to decide, and also
perform validations to ensure that the semantics of the parsed AST make sense.

When the AST node is visited, during semantic analyses, the type of the node
being a `Call` node will result in the dynamic method lookup feature of C++ to
call:

```{.cpp include=src/bpftrace-269/src/ast/semantic_analyser.cpp startLine=110 endLine=110}
```

This function will ensure that the vargs are visited first, if provided:

```{.cpp include=src/bpftrace-269/src/ast/semantic_analyser.cpp startLine=112 endLine=116}
```

Then the rest of the function is basically a big `switch` on `call.func`. In
order to handle a new function, we just have to add to the if ladder a case for
it:

```{.cpp include=src/bpftrace-269/src/ast/semantic_analyser.cpp startLine=216 endLine=225}
```

This describes what the function signature ought to be, and asserts as much by
examining the parsed AST node.

It also sets `call.type`, which can be thought of like the "return type" for
this builtin function. We can see that a new type, `Type::inet`, is added, that
indicates what is returned from this function.

While this PR only implemented 4 byte IPv4 addresses, it specifies specifies
that the arguments it takes will be in 3 x 64 bit array, so that it knows how
to access both IPv4 and, in the future, IPv6 addresses.

### Code Generation

Now that the function has been described, I had to implement the code to that
would actually get compiled to eBPF. This is a sort of meta-programming, where
you call LLVM functions to build LLVM IR. The level of this programming is very
much like assembly, but genericized. LLVM optimizes this IR (intermediate
representation), to output the resulting eBPF probes that con be loaded into
the kernel.

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=411 endLine=424}
```

We declare the allocation with `CreateAllocaBPF` noting the `inet` type:

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=413 endLine=415}
```

Then we actually inject a `memset` instruction, to actually allocate an array
of the specified size of a 3 x 64 bit array (24 bytes):

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=413 endLine=415}
```
Next, we want to be able to reference specific values within this array as we
described in the signature of the semantic analyser:

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=417 endLine=418}
```

This command, `CreateGEP` is like asking to get the pointer within the above
allocated array buffer. We can see that since the `af_offset` is 0, it is the
first byte. To get the IPv4 address, we get the value in the second byte, by
offsetting the first 8 bytes we already read, and get this `inet_offset`.

To actually read these values, we visit the code generation for the first arg,
at which point we know that the the global `expr_` has been set by the accept
call to the type of the AST node for the first argument. 

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=419 endLine=420}
```

Since we validated that this must be of `Type::integer` above in the semantic
analyzer, we can see that this is the function that will get called and how it
sets `expr_` to what we want to store:

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=18 endLine=21}
```

Then, this call to `CreateStore` says to copy the data from the address of the
first argument, to the location we pointed to for `af_offset` above. This is
actually storing the value in our buffer:

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=420 endLine=420}
```
We repeat this process to also get the IPv4 address:

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=421 endLine=422}
```

Then, following the same convention, we set `expr_` to be the 3 x 64 bit buffer
that we allocated, in a sense, "returning" it in the way that the
`Type::integer` was "returned".

Some parts of bpftrace also need to be made aware that rather than "returning"
a single, 64 bit value, it is returning an "array" of these, and so should be
treated like other "arrays", such as `usym` and `string` types in bpftrace:

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=1190 endLine=1197}
```

```{.cpp include=src/bpftrace-269/src/ast/codegen_llvm.cpp startLine=1221 endLine=1228}
```

### Calling inet_ntop

Now that all the work has been done to prepare the necessary bytes by
allocating some space in a predictable schema for the arguments, the actually
call to `inet_ntop` can be made!

A new function is implemented, that accepts this 3 x 64 bit array we have
described in the semantic analyzer, and implemented in LLVM code generation:
`

```{.cpp include=src/bpftrace-269/src/bpftrace.cpp startLine=1347 endLine=1360}
```

This function will be called each time a new IPv4 address needs to be parsed.
It indicates that IPv6 addresses are not yet supported, but if the `AF_INET`
type is as expected, it will parse the address using the library function
`inet_ntop`, then return the string representation of this value.

That concludes the work that was done to get support IPv4 network addresses! As
this wis my main (and probably the main) use-case, IPv6 support could come
later.

### Improvements to network tracing

The glaring gap of my contribution was that it didn't handle IPv6 addresses,
because at the time there was no way to access this data, as the struct parsing
of bpftrace was limited in determining the offsets of IPv6 fields.

This is not an issue when using later kernel versions, which can use the
IP tracepoint API to avoid this problem altogether.

// FIXME - discuss how mmarchini further extended this for IPv6

