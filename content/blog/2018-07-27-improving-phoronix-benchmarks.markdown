Title: Improving Phoronix Benchmarks
Date: 2018-07-27 22:22
Authors: Rashmica Gupta, Daniel Black, Anton Blanchard, Nick Piggin
Category: Performance
Tags: performance, phoronix, benchmarks
 

### Intro

Recently Phoronix ran a range of
[benchmarks](https://www.phoronix.com/scan.php?page=article&item=power9-talos-2&num=1)
comparing the performance of our POWER9 processor against the Intel Xeon and AMD
EPYC processors. 

We did well in the Stockfish, LLVM Compilation, Zstd compression, and the
Tinymembench benchmarks. A few of my colleagues did a bit of investigating into
the benchmarks where we didn't perform quite so well.

Some of the benchmarks where we don't perform as well as Intel are where the
benchmark has inline assembly for x86 but uses generic C compiler generated
assembly for POWER9. We also found a few things that should result in better
performance for all architectures.


### LBM / Parboil 

The [Parboil benchmarks](http://impact.crhc.illinois.edu/parboil/parboil.aspx) are a
collection of programs from various scientific and commercial fields that are
useful for examining the performance and development of different architectures
and tools.  Phoronix uses the lbm
[benchmark](https://www.spec.org/cpu2006/Docs/470.lbm.html): a fluid dynamics
simulation using the Lattice-Boltzmann Method.


There were a couple of issues with this benchmark:

1. The benchmark is compiled without any optimisation. Adding -O3 improves the
   result 3.2x. Adding optimisation will improve x86_64, but we expect it to
improve POWER9 results a lot more.


2. Each time step has to complete before the next one can start.  Unfortunately
   the resolution is quite small, which means there is very little work to do in
each time step. Therefore there is a limit to how many CPUs we can scale to,
there just isn't enough work.

This is a very old benchmark, and really it should be resized to suit modern
computers (eg increase resolution).


### x264 Video Encoding
x264 is a library that encodes videos into the H.264/MPEG-4 format. x264 encoding
requires a lot of integer kernels doing operations on image elements. The math
and vectorisation optimisations are quite complex, so Nick only had a quick look at
the basics.  The systems and environments (e.g. gcc version 8.1 for Skylake, 8.0
for POWER9) are not completely apples to apples so for now patterns are more
important than the absolute results. Interestingly the output video files between
architectures are not the same. All tests were run single threaded to
avoid any SMT (simulatenous multi-threading) effects.

Skylake is significantly faster than POWER9 on this test
(Skylake: 9.20 fps, POWER9: 3.39 fps).

One might hazard a guess that we could get better results by writing powerpc specific
assembly for more functions (in the version of x264 that Phoronix used there are only
six powerpc specific files compared with 21 x86 specific files). When running
with only the common C code (using the configure option --disable-asm) we have about
same performance (Skylake: 1.47 fps, POWER9: 1.54 fps). However even without the architecture
specific asm, the output files are not identical between CPUs or
different compile options, which is a concern with no floating point. Perhaps
there is a bug or some undefined behaviour in the code?

x264 compiles with -fno-tree-vectorize by default, which disables auto
vectorization. Removing this option along with link time optimisations
gives: Skylake: 2.25 fps,  POWER9: 3.02 fps. 

Now let's turn asm optimised code back on (removing the --disable-asm option).
A costly function, quant_4x4x4 is not vectorized in the POWER9 code. Without
-fno-tree-vectorize and with a small change to the code gcc can automatically 
vectorize it for us, giving up a slight speedup: Skylake: 9.20 fps, POWER9: 3.83 fps.

Even trying to account for the different vector sizes by restricting Skylake
to SSE4.2 with 128 bit vectors (same as POWER9) we still aren't even close to
Skylake: Skylake: 8.37 fps, POWER9: 3.83 fps.

| Test |Skylake |POWER9|
|-------|---------|-------|
| Original |9.20 fps |3.39 fps|
| Common C code only |1.47 fps | 1.54 fps|
| Common C code and autovectorisation enabled| 2.25 fps | 3.02 fps| 
| Arch specific asm and autovectorisation enabled| 9.20 fps | 3.83 ps |
| Arch specific asm, autovectorisation enabled, same vector sizes | 8.37 fps | 3.83 fps|

So we probably need some more powerpc specific implementations. If you're interested in looking into this, we do have some 
[active bounties](https://www.bountysource.com/teams/ibm/bounties) for x264 (lu-zero/x264).

### Primesieve

[Primesieve](https://primesieve.org/) is a program and C/C++
library that generates all the prime numbers below a given number. It uses an
optimised [Sieve of Eratosthenes](https://upload.wikimedia.org/wikipedia/commons/b/b9/Sieve_of_Eratosthenes_animation.gif)
implementation.

The algorithm uses the L1 cache size as the sieve size for the core loop.  This
is an issue when we are running in SMT mode (aka more than one thread per core)
as all threads on a core share the same L1 cache and so will constantly be 
invalidating each others cache-lines. As you can see
in the table below, running the benchmark in single threaded mode is 30% faster
than in SMT4 mode!

This means in SMT-4 mode the workload is about 4x too large for the L1 cache.  A
better sieve size to use would be the L1 cache size / number of
threads per core. Anton posted a [pull request](https://github.com/kimwalisch/primesieve/pull/54) 
to update the sieve size.

Interesting that the best overall performance on POWER9 is with the patch applied and in
SMT2 mode:

|SMT level   |    baseline   |     patched|
|------------|---------------|----------|
|1           |    14.728s     | 14.899s	|
|2           |    15.362s     | 14.040s	|
|4           |    19.489s     | 17.458s	|


### LAME 

Despite its name, a recursive acronym for "LAME Ain't an MP3 Encoder",
[LAME](http://lame.sourceforge.net/) is indeed an MP3 encoder.

Due to configure options [not being parsed correctly](https://sourceforge.net/p/lame/mailman/message/36371506/) this
benchmark is built without any optimisation regardless of architecture. We see a
massive speedup by turning optimisations on, and a further 6-8% speedup by
enabling
[USE_FAST_LOG](https://sourceforge.net/p/lame/mailman/message/36372005/) (which
is already enabled for Intel).

| LAME | Duration | Speedup |
|---------|-------------|--|
| Default | 82.1s | n/a |
| With optimisation flags | 16.3s | ~5.0x |
| USE_FAST_LOG set | 15.6s | ~5.3x  |

For more detail see Joel's
[writeup](https://shenki.github.io/LameMP3-on-Power9/).



### FLAC

[FLAC](https://xiph.org/flac/) is an alternative encoding format to
MP3. But unlike MP3 encoding it is lossless!  The benchmark here was encoding
audio files into the FLAC format. 

The key part of this workload is missing
vector support for POWER8 and POWER9. Anton and Amitay submitted this
[patch series](http://lists.xiph.org/pipermail/flac-dev/2018-July/006351.html) that
adds in POWER specific vector instructions. It also fixes the configuration options
to correctly detect the POWER8 and POWER9 platforms. With this patch we get see about a 3x
improvement in this benchmark.


### OpenSSL

Mainline OpenSSL is almost 2x faster (than the version of OpenSSL that phoronix used??) because
of [this commit](https://github.com/openssl/openssl/commit/68f6d2a02c8cc30c5c737fc948b7cf023a234b47).
This commit adds some optimised multiplication and squaring assembly code.
Other than that, it's mostly hardware.


### SciKit-Learn

SciKit-Learn is a bunch of python tools for data mining and
analysis (aka machine learning).

Joel noticed that the benchmark spent 92% of the time in libblas. Libblas is a
very basic BLAS (basic linear algebra supprograms) library that python-numpy
uses to do vector and matrix operations.  The default libblas on Ubuntu is only
compiled with -O2. Compiling with -Ofast and using alternative BLAS's that have
PowerPC optimisations (such as libatlas or libopenblas) we see big improvements
in this benchmark (these times are for one run of computations, the benchmark
does this 5 times):


| BLAS used | Duration | Speedup |
|---------|-------------|--|
| libblas -O2 |64.2s | n/a |
| libblas -Ofast |  36.1s | ~1.8x |
| libatlas | 8.3s | ~7.7x  |
|libopenblas | 4.2s | ~15.3x |


You can read more details about this
[here](https://shenki.github.io/Scikit-Learn-on-Power9/).


### Blender

Blender is a 3D graphics suite that supports image rendering,
animation, simulation and game creation. On the surface it appears that Blender
2.79b (the distro package version that Phoronix used by system/blender-1.0.2)
failed to use more than 15 threads, even when "-t 128" was added to the bender
command line.

It turns out that even though this benchmark was supposed to be run on CPUs only
(you can choose to render on CPUs or GPUs), the GPU file was always being used.
The GPU file is configured with a very large tile size of 256x256 being used -
which is (fine for
GPUs)[https://docs.blender.org/manual/en/dev/render/cycles/settings/scene/render/performance.html#tiles]
but not great for CPUs. The image size (1280x720) to tile size ratio limits the
number of jobs created and therefore the number threads used. Fortunately this
has already been fixed in the
[pts/blender-1.1.1](https://openbenchmarking.org/test/pts/blender) of Phoronix.

https://github.com/phoronix-test-suite/test-profiles/issues/24 

To obtain a realistic CPU measurement with more that 15 threads you can force
the use of the cpu file by overwritting the gpu file with the cpu one:

```$ cp
~/.phoronix-test-suite/installed-tests/system/blender-1.0.2/benchmark/pabellon_barcelona/pavillon_barcelone_cpu.blend
~/.phoronix-test-suite/installed-tests/system/blender-1.0.2/benchmark/pabellon_barcelona/pavillon_barcelone_gpu.blend```

As you can see in the image below, now all of the cores are being utilised!
![Blender HTOP Output][00]



| Benchmark | Duration (deviation over 3 runs) | Speedup |
|--------------------------------|------------------------------|---------|
|Baseline (GPU blend file) |  1509.97s (0.30%) | n/a |
| Single 22-core POWER9 chip (CPU blend file)  | 458.64s (0.19%) | 3.29x  |
| Two 22-core POWER9 chips (CPU blend file)  | 241.33s (0.25%) | 6.25x |


Pinning the pts/bender-1.0.2, Pabellon Barcelona, CPU-Only test to a single
22-core POWER9 chip (```sudo ppc64_cpu --cores-on=22```) and two POWER9 chips
(```sudo ppc64_cpu --cores-on=44```) show a huge speedup.

[00]: /images/phoronix/blender-88threads.png "Blender with CPU Blend file"



