---
layout: page
title: About
permalink: /about/
---

The SIMDe header-only library provides fast, portable implementations of
[SIMD intrinsics](https://en.wikipedia.org/wiki/SIMD) on hardware which
doesn't natively support them, such as calling [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
functions on ARM.  There is no performance penalty if the hardware
supports the native implementation (*e.g.*, SSE/[AVX](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions)
runs at full speed on [x86](https://en.wikipedia.org/wiki/X86),
[NEON](https://en.wikipedia.org/wiki/ARM_architecture#Advanced_SIMD_(Neon)) on [ARM](https://en.wikipedia.org/wiki/ARM_architecture),
*etc.*).
