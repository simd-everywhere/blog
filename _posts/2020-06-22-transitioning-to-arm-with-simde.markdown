---
layout: post
title: "Transitioning SSE/AVX code to NEON with SIMDe"
date: 2020-06-22 12:28:03 -0700
tags: sse avx neon apple
author: Evan Nemerson
---

Now that Apple [has
announced](https://arstechnica.com/gadgets/2020/06/this-is-apples-roadmap-for-moving-the-first-macs-away-from-intel/)
that they will be moving away from x86 to their own ARM-based CPUs,
lots of people will be stuck with SIMD code targeting x86 ISA
extensions like SSE, SSE2, AVX, etc., which won't run on Apple's new
machines.

Arm CPUs do have support for SIMD, but instead of Intel technologies
like SSE and AVX, Arm has
[NEON](https://developer.arm.com/architectures/instruction-sets/simd-isas/neon).
NEON is an improvement over the x86 APIs in a lot of ways, and a
regression in others, but it is undeniably *different* and you can't
just recompile your application on Arm and expect it to work.

Or can you?  SIMD Everywhere (SIMDe) provides fast, portable,
permissively-licensed (MIT) implementations of the x86 APIs which
allow you to run code designed for x86/x86_64 CPUs pretty much
anywhere, including on Arm (using NEON if available).  With almost no
source code changes, you can recompile your x86 SIMD code for Arm (or
POWER, or WebAssembly, etc.).

If NEON is available, SIMDe will even use it to provide the x86
functions.  For example, `_mm_add_ps` from SSE can be implemented
using NEON's `vaddq_ps` function, so that's exactly what SIMDe does.
For more complicated functions without direct analogs in NEON SIMDe
will use the fastest implementation we can.  Hopefully that means
calling multiple NEON functions, but in the worst case scenario SIMDe
has completely portable C99 fallbacks.

If you'd like to take SIMDe for a test drive, it is usable [on
Compiler Explorer](https://godbolt.org/z/wf4t42).  Compilation is a
bit slow, due to having to transfer large files, but it's quite
usable.

Before I continue, it's worth noting that SIMDe has [an active chat
room](https://gitter.im/simd-everywhere/community), [a less-active
mailing list](https://groups.google.com/forum/#!forum/simde), and [a
very active issue
tracker](https://github.com/simd-everywhere/simde/issues) where
questions are welcome.  If you have any questions, problems, concerns,
etc., please get in touch!

## Getting SIMDe

I mentioned earlier that "almost no source code changes" are required.
Many of you are probably worried about the word "almost", so let's
discuss that for a bit.

First, you'll need to get SIMDe.  If you're on Debian there is a
[libsimde-dev](https://packages.debian.org/bullseye/libsimde-dev)
package, or on Fedora/Red Hat/etc. there is a
[simde](https://koji.fedoraproject.org/koji/packageinfo?packageID=31156)
package.  Both are pretty new, though, so they may not be available to
you yet.

If that doesn't work for you, you can drop a copy of SIMDe into your
project.  If you want to use a git submodule that will work, but the
[main repository](https://github.com/simd-everywhere/simde) is pretty
big thanks to all the tests.  If you want something a bit smaller we
also have a [simde-no-tests
repository](https://github.com/simd-everywhere/simde-no-tests/) which
is basically a mirror of only the implementations that is updated
automatically whenever SIMDe is updated.

SIMDe is a header-only library, and doesn't require any build system
integration; simply including the relevant headers is enough.  That
said, we do recommend aggressive optimizations (like `-O3`), and
enabling OpenMP SIMD (which does not introduce a run-time dependency
on OpenMP) with `-fopenmp-simd` on GCC and clang, or `-qopenmp-simd`
on ICC.  If you do enable OpenMP SIMD, please let SIMDe know by also
passing `-DSIMDE_ENABLE_OPENMP` (not necessary if you enable *full*
OpenMP, i.e. `-fopenmp` instead of `-fopenmp-simd`).

## Source-level changes

As far as source-level changes are concerned, all you need to do is
define the `SIMDE_ENABLE_NATIVE_ALIASES` macro, and include a SIMDe
header instead of *mmintrin.h.  SIMDe headers are named according to
the ISA extension they supply, so you don't need to remember which
letter corresponds to which ISA extension when writing your code, but
if you already have *mmintrin.h scattered around here is how they map
to SIMDe headers:

 * mmintrin.h → simde/x86/mmx.h
 * xmmintrin.h → simde/x86/sse.h
 * emmintrin.h → simde/x86/sse2.h
 * pmmintrin.h → simde/x86/sse3.h
 * tmmintrin.h → simde/x86/ssse3.h
 * smmintrin.h → simde/x86/sse4.1.h
 * nmmintrin.h → simde/x86/sse4.2.h

Starting with AVX Intel started using immintrin.h to just include
everything, so if you're using immintrin.h just include the header
for the "greatest" ISA extension you use; for example, if you want
AVX-512F, include simde/x86/avx512f.h.

Let's take a look at that `SIMDE_ENABLE_NATIVE_ALIASES` macro.  If you
don't define it, SIMDe will only define functions in its own `simde_*`
namespace.  For example, instead of `_mm_add_ps` you would need to use
`simde_mm_add_ps`.  If you *do* define `SIMDE_ENABLE_NATIVE_ALIASES`,
SIMDe will also use a function-like macro to create an alias:

```c
#define _mm_add_ps(a, b) simde_mm_add_ps(a, b)
```

While that works *most* of the time, there are a few things you'll
want to be aware of.  Perhaps the biggest problem is that Intel
doesn't use fixed-width types (`int8_t`, `int32_t`, `uint16_t`, etc.)
in their APIs, they instead assume specific characteristics of
standard types which are true on x86 but may not be true on other
platforms.  For example, on many Arm platforms, `char` is *unsigned*,
but Intel uses `char` to represent a *signed* 8-bit integer.

SIMDe deals with this by using fixed-width types in our
implementations so they work eveywhere, but if your code is using
`char` to mean `signed 8-bit integer` you may encounter problems when
attempting to use SIMDe functions on some platforms.  The good news is
that you can generally just change your code to use `int8_t` instead
of `char`; it will work exactly the same on x86 (`int8_t` is likely
just a typedef to `char`), and it will also work on other platforms.

## Completeness

SIMD APIs are big.  Very big.  x86/x86_64 alone currently has a bit
over 6,000 functions, of which SIMDe has implemented around 2,000.

That said, most of those are AVX-512F extensions which aren't widely
used yet.  Odds are quite good that you're only using extensions for
which SIMDe already has complete support (as of v0.5.0, released
2020-06-22):

 * [MMX](https://en.wikipedia.org/wiki/MMX_(instruction_set))
 * [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
 * [SSE2](https://en.wikipedia.org/wiki/SSE2)
 * [SSE3](https://en.wikipedia.org/wiki/SSE3)
 * [SSSE3](https://en.wikipedia.org/wiki/SSSE3)
 * [SSE4.1](https://en.wikipedia.org/wiki/SSE4#SSE4.1)
 * [AVX](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions)
 * [FMA](https://en.wikipedia.org/wiki/FMA_instruction_set)
 * [GFNI](https://en.wikipedia.org/wiki/AVX-512#GFNI)

We also have a very good start on many other extensions, including
AVX2, AVX-512F, AVX-512BW, AVX-512VL, and NEON (portable
implementations of NEON that can run on x86, or anywhere else).  Also,
it's not really a CPU extension, but our implementation of SVML is
coming along nicely.

If SIMDe is missing a particular function you need, please [file an
issue](https://github.com/simd-everywhere/simde/issues) and we may be
able to prioritize an implementation.  We're planning to implement all
functions anyways, and doing so in a slightly different order doesn't
generally create any extra work, so if it would help your project
we're generally happy to oblige.  Of course, if you're interested in
implementing something yourself instead of waiting for us patches are
always welcome!

## Debugging

SIMDe can be a fantastic tool for debugging.  Not only can you see
inside of the function to understand how it really works, you can also
run the code on your development machine in your native environment
*without an emulator*.  Obviously you'll eventually want to check
everything at least in an emulator, or preferably on real hardware,
but during development SIMDe can be immensely helpful.

## Performance

Honestly, it's pretty good.  When there is a NEON function that
implements exactly the same functionality there is no cost for using
SIMDe instead of calling the NEON function directly; the compiler
translates it to exactly the same code.

Even when we hit a portable fallback, the compiler is often smart
enough to auto-vectorize the code, especially if you have aggressive
optimizations (think `-O3`) enabled.  We use compiler-specific
functionality like [GCC-style vector
extensions](https://gcc.gnu.org/onlinedocs/gcc/Vector-Extensions.html)
(supported by pretty much every compiler except for MSVC), builtins
like
[`__builtin_shufflevector`](http://clang.llvm.org/docs/LanguageExtensions.html#langext-builtin-shufflevector),
[`__builtin_shuffle`](https://gcc.gnu.org/onlinedocs/gcc/Vector-Extensions.html#Vector-Extensions),
[`__builtin_convertvector`](http://clang.llvm.org/docs/LanguageExtensions.html#langext-builtin-convertvector),
etc. wherever possible, which generaly results in optimal
implementations.  Even when we hit portable fallbacks, they are
decorated with pragmas from OpenMP 4 SIMD, Cilk+, or copmiler-specific
hints like [GCC loop-specific
pragmas](https://gcc.gnu.org/onlinedocs/gcc/Loop-Specific-Pragmas.html)
or [clang pragma loop hint
directives](http://llvm.org/docs/Vectorizers.html#pragma-loop-hint-directives).
We try very hard to make sure that even the fallbacks are fast.

SIMDe will never make your project slower, only more portable.
Performance likely won't be as good as a manual rewrite by someone who
knows NEON well, but you can get a port up and running at almost no
cost in terms of developer resources, and once it's done you're free
to mix, for example, SSE and NEON code at will.  That means you can
gradually port specific portions of your code which are particularly
hot, or where SIMDe doesn't do a good job (though in that case please
[file an issue](https://github.com/simd-everywhere/simde/issues) too),
while leaving areas where SIMDe performance is adequate alone instead
of wasting development time and resources.

## Still have questions?

The [F.A.Q.](https://github.com/simd-everywhere/simde/wiki/FAQ) has
some information which may help.

If that doesn't answer your question please feel free to ask in [our
chat room](https://gitter.im/simd-everywhere/community), on [our
mailing list](https://groups.google.com/forum/#!forum/simde), or on
[our issue tracker](https://github.com/simd-everywhere/simde/issues);
if you have questions our documentation hasn't answered, it's a bug in
our documentation, so don't worry about using the issue tracker!
