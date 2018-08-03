Title: Improving Phoronix Benchmarks
Date: 2018-07-27 22:22
Authors: Rashmica Gupta
Category: Performance
Tags: performance, phoronix, benchmarks
 

### Intro

Recently Phoronix ran a range of [benchmarks](https://www.phoronix.com/scan.php?page=article&item=power9-talos-2&num=1) comparing the performance of our POWER9 processor against the Intel Xeon and AMD EPYC processors. 

We did well in the Stockfish, LLVM Compilation, Zstd compression, and the Tinymembench benchmarks. Some of my colleagues did a bit of investigating into the benchmarks where we didn't perform quite so well.

Some of the benchmarks where we don't perform as well as Intel are where the benchmark has inline assembly for x86 but uses generic C compiler generated assembly for POWER9. We also found a few things that should result in better performance for all architectures.

### Benchmark

Context - what can the sw be used for?
Problem - what is making the bench mark slow?
Changes - what did we change/fix?
Results - what are the speedups that we saw?


### LBM / Parboil
The [Parboil benchmarks](http://impact.crhc.illinois.edu/parboil/parboil.aspx) are a collection of programs from various scientific 
and commercial fields that are useful for examining the performance and development of different architectures and tools.
Phoronix uses the lbm [benchmark](https://www.spec.org/cpu2006/Docs/470.lbm.html): a fluid
dynamics simulation using the Lattice-Boltzmann Method.


There were a couple of issues with this benchmark:

1. The benchmark is compiled without any optimisation. Adding -O3 improves the result 3.2x. Adding
optimisation will improve x86_64, but we expect it to improve POWER9 results a lot more.


2. Each time step has to complete before the next one can start.
Unfortunately the resolution is quite small, which means there is very
little work to do in each time step. Therefore there is a limit to how
many CPUs we can scale to, there just isn't enough work.

This is a very old benchmark, and really it should be resized to suit
modern computers (eg increase resolution).



### x264 Video Encoding
x264 is a library that encodes videos into the H.264/MPEG-4 format.

Mostly vectorisation issues. We have [active bounties](https://www.bountysource.com/teams/ibm/bounties) for x264.



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
[Primesieve](https://primesieve.org/) is a program and C/C++ library that generates all the prime numbers below a given number. It uses an optimised [Sieve of Eratosthenes](https://upload.wikimedia.org/wikipedia/commons/b/b9/Sieve_of_Eratosthenes_animation.gif) implementation.

The algorithm uses the L1 cache size to size the core loop.
It correctly calculates the L1 size, but then doesn't realise that SMT
threads share a core. This means the workload is 4x too large for the
L1 cache.

Running it in ST mode gives a 30% improvement. We could ask them to
rerun in ST mode, or perhaps look at teaching primesieve about
threading.

Anton posted a [first pass](https://github.com/kimwalisch/primesieve/pull/54) at this.

Interesting that the best overall performance is with the patch applied
and in SMT2 mode.

Time in seconds:

|SMT level   |    baseline   |     patched|
|------------|---------------|-----------|
|1           |    14.728     |     14.899|
|2           |    15.362     |     14.040|
|4           |    19.489     |     17.458|

### FLAC
[FLAC](https://xiph.org/flac/) is an alternative encoding format to MP3. But unlike MP3 encoding it is lossless!
The benchmark here was encoding audio files into the FLAC format. The key part of this workload is missing vector support for POWER8 and POWER9. With this [patch series](http://lists.xiph.org/pipermail/flac-dev/2018-July/006351.html) we get at best a 3.3x improvement.

### LAME
Despite its name, a recursive acronym for "LAME Ain't an MP3 Encoder", [LAME](http://lame.sourceforge.net/) is indeed an MP3 encoder.

Joel already wrote about the improvements [here](https://shenki.github.io/LameMP3-on-Power9/) but the gist is that this benchmark is built without any optimisation regardless of architecture. We see a massive speedup by turning optimisations on, and a further 8% speedup by enabling FAST_LOG (which is already enabled for Intel).



### OpenSSL

Mainline OpenSSL is almost 2x faster (than the version phoronix used??) because of [this commit](https://github.com/openssl/openssl/commit/68f6d2a02c8cc30c5c737fc948b7cf023a234b47). This commit adds some optimised multiplication and squaring assembly code. 
Other than that, it's mostly hardware.


### SciKit-Learn
SciKit-Learn is a bunch of tools for data mining and analysis (aka machine learning).

This benchmark uses libblas, a very basic BLAS based on Fortran. It only uses 1 CPU, and even worse Ubuntu/Debian (worse than what?). Using alternative BLAS's that have PowerPC optimisations (such as libatlas or libopenblas) we see big improvements in this benchmark.

- decided to disable vectorisation on ppc64le only.

Joel also wrote up his work on this [here](https://shenki.github.io/Scikit-Learn-on-Power9/).

### Blender
Blender is a 3D graphics suite that supports animation, simulation, game creation and more! On the surface it appears that Blender 2.79a (the version that Phoronix used) has some multithreading issues - it doesn't use more than 15 threads even when using -t 128.


However it turns out that even though this benchmark was supposed to be run on CPUs only (you can choose to render on CPUs or GPUs), the GPU file was always being used. This results in a very large tile size of 256x256 being used - which is (fine for GPUs)[https://docs.blender.org/manual/en/dev/render/cycles/settings/scene/render/performance.html#tiles] but not great for CPUs. The image size to tile size ratio limits the number of jobs created and therefore the number threads used. Fortunately this has already been fixed in the <version> of phoronix.

Or you can hack it by overwriting the gpu file with the cpu one:

```phoronix-test-suite$ cp installed-tests/system/blender-1.0.2/benchmark/pabellon_barcelona/pavillon_barcelone_cpu.blend  installed-tests/system/blender-1.0.2/benchmark/pabellon_barcelona/pavillon_barcelone_gpu.blend```

Speedup of ~XXX.
