# Adding ntop to bpftrace

In order to debug network traffic and implement tcp tracing scripts for
bpftrace, the main thing I knew was missing was the ability to print IP
addresses. I decided to tackle the existing issue for implementing ntop
[@bpftrace-issue-30]. So I set to work on this, which was my first introduction
to bpftrace, and got to work on [@bpftrace-pull-269].

## Lexer and Parser

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

## Semantic Analyser

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

## Code Generation

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

## Calling inet_ntop

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

## tcp tools enabled by ntop support

I was able to use this functionality to port over IPv4 versions of the missing
tcp tools! Exactly what I had wanted to do. Using basically the same approach,
I was able to implement simple versions of:

* tcpaccept.bt: a bpftrace version of tcpaccept.py, which traces when a a new
tcp connection has been accepted, great for checking what is connecting to a
server.
* tcpconnect.bt: a bpftrace version of tcpconnect.py, which traces outgoing
tcp connections, to see what a server is connection to for outbound connections.
* tcpdrop.bt: a bpftrace version of tcpdrop.py, which traces when tcp packets
are dropped by the linux kernel for some reason or another, useful for checking
if the network is behaving reliably.
* tcpretrans.bt: a bpftrace version of tcpretrans.py, which traces when tcp
packets need to be retransmitted by the linux kernel, useful for checking if
the network is behaving reliably.

These all hooked into the same places their bcc counterparts did, and some were
limited by the current functionality of bpftrace at the time, as the header
parsing was limited in what could be accessed via dynamic tracing, and many tcp
tracepoints weren't widely available yet.

Many of these scripts didn't adhere to best practices for style, and others
have graciously improved on them and kept them up-to-date with the other
bpftrace scripts, ensuring they adhere to bpftrace idioms.

// FIXME attribution to these fine folks
