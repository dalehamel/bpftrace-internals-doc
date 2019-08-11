# Improvements ntop

The glaring gap of my contribution was that it didn't handle IPv6 addresses,
because at the time there was no way to access this data, as the struct parsing
of bpftrace was limited in determining the offsets of IPv6 fields.

This is not an issue when using later kernel versions, which can use the
IP tracepoint API to avoid this problem altogether. Support was then added by
Matheus Marchini [@mmarchini] to refactor this to handle IPv6 addresses
[@bpftrace-pull-566] and it is instructive to examine these changes.

I did not actually do this work, but I followed it and felt impelled to
describe the improvements Matheus made, as I also reviewed his pull request.

## Semantic analyser

In the semantic analyser, Matheus updated the implementation:

```{.cpp include=src/bpftrace-566/src/ast/semantic_analyser.cpp startLine=313 endLine=345}
```

Note that it better describes the storage of the argument as a struct, and how
the bytes are intended to be used to describe the union of the addresses.

He also allowed for different argument types to be accepted, and for the buffer
size to be used to be specified dynamically.

This allows for the signature of ntop to be more complicated, with default and
variable number arguments.

## Code generation

The new implementation of the code generation must correspondingly handle the
new cases, for both IPv4 and IPv6 addresses.

```{.cpp include=src/bpftrace-566/src/ast/codegen_llvm.cpp startLine=477 endLine=515}
```

He updated the integer type to support 128 bit types, as would be needed for
IPv6, meaning that if a 128 bit integer could be read (such as by a tracepoint)
it could hold a literal IPv6 value!

```{.cpp include=src/bpftrace-566/src/ast/irbuilderbpf.cpp startLine=111 endLine=115}
```

This was a trick I had discovered too, but hadn't committed as I couldn't find
a good case for supporting 128 bit ints generically in bpftrace. // FIXME link issue.

Since an array can also be passed, we determine if the array holds 4 bytes
(IPv4) or 16 bytes (IPv6). We also use 4 bytes to store AF_INET. This is why we
either have a buffer size of 4 bytes (AF_INET) + 4 bytes (IPv4) = 8 bytes for
IPv4 addresses, or 4 bytes (AF_INET) + 16 bytes (IPv6) = 20 bytes for IPv6
addresses.

We accordingly make the `af_inet` determination based on the buffer size and
number of arguments passed:

```{.cpp include=src/bpftrace-566/src/ast/codegen_llvm.cpp startLine=491 endLine=503}
```

Since the address is always after the first 4 bytes, we can just read the rest
of the bytes handling both the literal integer, and array cases:

```{.cpp include=src/bpftrace-566/src/ast/codegen_llvm.cpp startLine=505 endLine=512}
```

## Calling inet_ntop

Now that the codegen is implemented, we have to update our helper to support
calling the `inet_ntop` for both `AF_INET` and `AF_INET6` address types:

```{.cpp include=src/bpftrace-566/src/bpftrace.cpp startLine=1675 endLine=1688}
```

Which then calls either the IPv4 or IPv6 static / private versions accordingly:

```{.cpp include=src/bpftrace-566/src/bpftrace.cpp startLine=1661 endLine=1672}
```
