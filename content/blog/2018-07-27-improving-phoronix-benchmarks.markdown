Title: Improving Phoronix Benchmarks
Date: 2018-07-27 22:22
Authors: Rashmica Gupta
Category: Performance
Tags: performance, phoronix, benchmarks
 

### Intro


[Benchmarks](https://www.phoronix.com/scan.php?page=article&item=power9-talos-2&num=1)

### OpenMP / Parboil

### x264 Video Encoding
x264 has a lot of integer kernels doing operations on image elements. The math and vectorisation optimisations are quite complex, so I started looking at the basics first. I'm comparing a POWER9 vs Skylake workstation. The systems and environments (e.g. gcc version 8.1 for Skylake, 8.0 for POWER9) are not completely apples to apples so for now the absolute results don't matter so much as patterns. I'm doing all tests with --threads 1 to start with, to avoid SMT effects.

Skylake is significantly faster than POWER9 on this test:
Skylake: 9.20 fps
POWER9 : 3.39 fps

Let's start with a baseline, run both with --disable-asm that generates unvectorized binaries from common C code:
Skylake: 1.47 fps
POWER9 : 1.54 fps

Even with --disable-asm, the output files are not identical between CPUs or different compile options, which is a concern with no floating point. Perhaps there is a bug or some undefined behaviour in the code.

x264 compiles with -fno-tree-vectorize by default, which disables auto vectorization. Enabling it (plus LTO) gives:
Skylake: 2.25 fps
POWER9 : 3.02 ps

Now let's remove --disable-asm to turn asm optimised code back on. A costly function, quant_4x4x4 is not vectorized in the POWER9 code. Without -fno-tree-vectorize, gcc can vectorize it with a small change to the loop (output file is unchanged):
Skylake: 9.20 fps
POWER9 : 3.83 ps

Restricting Skylake to SSE4.2 with 128 bit vectors (same as POWER9) shows that AVX2 does't provide huge gains:
Skylake: 8.37 fps
POWER9 : 3.83 fps

Skylake is still much faster at the same vector size. It's surpising they are able to get 5.7x speedup with 128 bit vectors. 


### Primesieve

### FLAC

### Lame

### OpenSSL

### SciKit-Learn

### Blender
