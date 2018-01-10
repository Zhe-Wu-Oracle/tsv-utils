_Visit the [main page](../README.md)_

# Building with Link Time Optimization and Profile Guided Optimization

This page provides instruction for building the TSV utilities from source code using Link Time Optimization (LTO) and Profile Guided Optimization (PGO). LTO is enabled for all the tools, PGO is enabled for a select few. Both improve run-time performance, LTO has the additional effect of reducing binary sizes. Normally PGO and LTO can be used independently, however, the TSV utilities build system only supports PGO when already using LTO.

Contents:

  * [About Link Time Optimization](#about-link-time-optimization-lto)
  * [About Profile Guided Optimization](#about-profile-guided-optimization-pgo)
  * [Building the TSV utilities with LTO and PGO](#building-the-tsv-utilities-with-lto-and-pgo)
  * [Additional options](#additional-options)
  * [LDC command lines](#ldc-command-lines)
  
Skip down to [Building the TSV utilities with LTO and PGO](#building-the-tsv-utilities-with-lto-and-pgo) to get right to the build instructions.

## About Link Time Optimization (LTO)

Link Time Optimization is an approach to whole program optimization that performs additional optimizations during link-time that are impossible to do at compile time when only a part of the program is available. LDC, the LLVM-based D compiler, supports Link Time Optimization via LLVM's LTO.

When LTO is used, the compiler saves its intermediate representation code in `.o` files rather than machine code. In LLVM's case, LLVM bitcode. At link time the linker calls back into LLVM plugin modules that optimize code generation across the entire program. This enables optimizations that cross function and module boundaries (effectively, it's as if your whole program, including external libraries, were all in the same `.d` file). Cross-module inlining is an example of an optimization that can often be done quite effectively via LTO.

This is a powerful technique, but involves more complex cooperation between compiler and linker than the traditional compile-link cycle. It is only recently that LTO has started to become widely supported by software development toolchains.

LDC has supported LTO for several releases, however, only macOS was fully supported out-of-the-box. With the LDC 1.5.0 release, LTO is now available out-of-the-box on both Linux and macOS. Windows LTO support is in progress.

A valuable enhancement introduced in LDC 1.5.0 is support for compiling the D runtime library and standard library (Phobos) with LTO. This enables interprocedural optimizations spanning both D libraries and application code. For the TSV utilities this produces materially faster executables.

Compiling the D standard libraries with LTO is done using the `ldc-build-runtime` tool, included with the LDC 1.5.0 release. This tool downloads the source code for the D standard libraries and compiles it with user-specified compile flags. The `ldc-build-runtime` tool makes it easy to rebuild the D standard libraries with LTO enabled. These LTO compiled libraries can be included on the `ldc2` compile/link command when building the application for maximum LTO opportunities. Applications can also be built compiling just the application code with LTO, linking with the static versions of the D standard libraries shipped with LDC.

There are two different forms of LTO available: Full and Thin. To build the TSV utilities with LTO is sufficient to know that they exist and are incompatible with each other. For information on the differences see the LLVM blog post [ThinLTO: Scalable and Incremental LTO](http://blog.llvm.org/2016/06/thinlto-scalable-and-incremental-lto.html).

## About Profile Guided Optimization (PGO)

Profile Guided Optimization (PGO) is an approach to optimization based on recording typical execution patterns. Execution behavior data enables better choices for branch prediction, inlining decisions, and other optimizations.

There are several ways to gather execution behavior statistics, the approach used by the TSV utilities is to generate an instrumented build and run it on common inputs. The result of these runs is recorded and passed to the compiler and linker to generate the final executable.

The TSV utilities build system supports PGO for a couple tools, those showing the most benefit. Currently, PGO is enabled only when also using LTO. The source repository contains everything needed to generate profile data for the build, including data files and scripts invoking the instrumented builds.

For a more detailed introduction to PGO see [Profile-Guided Optimization with LDC](https://johanengelen.github.io/ldc/2016/07/15/Profile-Guided-Optimization-with-LDC.html) on Johan Engelen's blog.

## Building the TSV utilities with LTO and PGO

The pre-built binaries available from the [releases page](https://github.com/eBay/tsv-utils-dlang/releases) are compiled with LTO and PGO for both D libraries and the TSV utilities code. This is not enabled by default when building from source code. The reason is simple: LTO is still an early stage technology. Testing on a wider variety of platforms is needed before making it the default.

However, LTO and PGO builds can be enabled by setting makefile parameters. Testing with the built-in test suite should provide confidence in the resulting applications. The TSV utilities makefile takes care of invoking both `ldc-build-runtime` and `ldc2` with the necessary parameters.

**Prerequisites:**
  * LDC 1.5.0 or later. See the LDC project [README](https://github.com/ldc-developers/ldc/blob/master/README.md) for installation instructions.
  * TSV utilities source code, 1.15.0-beta3 or later (1.16.0 for PGO).
  * Linux or macOS. macOS requires Xcode 9.0.1 or later.

Linux builds have been tested on Ubuntu 14.04 and 16.04.

**Retrieve the TSV utilities source code:**

Via git clone:

```
$ git clone https://github.com/eBay/tsv-utils-dlang.git
$ cd tsv-utils-dlang
```

Via DUB (replace `1.1.15` with the version retrieved):

```
$ dub fetch tsv-utils-dlang --cache=local
$ cd tsv-utils-dlang-1.1.15
```

Via the source from the GitHub [releases page](https://github.com/eBay/tsv-utils-dlang/releases) (replace `1.1.15` with the latest version):

```
$ curl -L https://github.com/eBay/tsv-utils-dlang/archive/v1.1.15.tar.gz | tar xz
$ cd tsv-utils-dlang-1.1.15/tsv-utils-dlang
```

**Build with LTO enabled:**

```
$ make DCOMPILER=ldc2 LDC_BUILD_RUNTIME=1
$ make test-nobuild
```

The above command builds with LTO on both the D libraries and the TSV utilities code. The build should be good if the tests succeed.

**Build with LTO and PGO enabled:**

To use PGO, add either `LDC_PGO=1` or `LDC_PGO=2` to the above command:

```
$ make DCOMPILER=ldc2 LDC_BUILD_RUNTIME=1 LDC_PGO=1
$ make test-nobuild
```

This turns on PGO for the tools supporting it. The two values are used to help with build times.

- `LDC_PGO=1` - Enables PGO for those tools showing the largest performance gains.
- `LDC_PGO=2` - Enable PGO for all tools that have been configured to use it.

The PGO setup creates and runs an instrumented build to collect profiling data, this is what increases build times. Build times are still not excessive, but `LDC_PGO=1` can be a nice compromise.

## Additional options

The above instructions should be sufficient to create valid LTO builds. This section describes a few additional choices.

### Use LTO on the TSV utilities code only

The largest gains come from using LTO on both the D libraries and the application code. However, LTO can also be used on the application code alone. For the TSV utilities, this is the default on macOS. On Linux it needs to be enabled explicitly.

**macOS:**
```
$ make DCOMPILER=ldc2
$ make test-nobuild
```

**Linux:**
```
$ make DCOMPILER=ldc2 LDC_LTO=full
$ make test-nobuild
```

### Statically linking the C runtime library on Linux builds

The prebuilt Linux binaries statically link the C runtime library. This increases portability at the expense of increased binary sizes. The earlier instructions dynamically link the C runtime library. To use static linking, add `DFLAGS=-static` to the build lines, as follows:

```
$ make DCOMPILER=ldc2 LDC_BUILD_RUNTIME=1 DFLAGS=-static
$ make test-nobuild
```

This is only applicable to Linux builds, and has only been tested on Ubuntu. On Ubuntu, release 16.04 or later is required.

### Choosing between Thin and Full LTO

The makefile default settings choose between Thin and Full LTO builds based on the platform. At present, Thin is used on macOS, Full is used on Linux. This is based on the author's configuration testing. At the time this was written (LDC 1.5.0, TSV Utilities 1.1.15), issues have surfaced with other choices. These issues tend to be complex, involving a combination of LDC, LLVM, and the system linkers. These are likely to be fixed in future releases, but for now the default configurations are recommended. These problems surface primarily when building both D libraries and application code using LTO (`LDC_BUILD_RUNTIME=1`). Building with LTO applied to the application code alone works fine in the configurations tested. For this, use `LDC_LTO=thin` or `LDC_LTO=full`, without specifying the `LDC_BUILD_RUNTIME` parameter.

The `LDC_LTO=thin|full` parameter can also be combined with `LDC_BUILD_RUNTIME=1` to set the Thin/Full type when building the D libraries with LTO.

It is also possible to turn off LTO on macOS builds. For this use `LDC_LTO=off`.

## LDC command lines

Running the `make` commands shown above will display the LDC command lines. They are a bit lengthy though. The examples below show the command lines for building a simple `helloworld` program with LTO enabled. See the [LDC documentation](https://github.com/ldc-developers/ldc) for up-to-date details. See the LDC documentation or  [Profile-Guided Optimization with LDC](https://johanengelen.github.io/ldc/2016/07/15/Profile-Guided-Optimization-with-LDC.html) (Johan Engelen's blog) for PGO build parameters.

There are two steps for building with LTO. The first is downloading and building the D library code, the second is to reference the LTO built library from the application build command.

**Download and build the D library code:**

```
$ # Build with Thin LTO
$ ldc-build-runtime --reset --dFlags="-flto=thin" BUILD_SHARED_LIBS=OFF

$ # Build with Full LTO
$ ldc-build-runtime --reset --dFlags="-flto=full" BUILD_SHARED_LIBS=OFF
```

This builds in the `ldc-build-runtime.tmp` directory. The `--reset` option avoids downloading the source code if it's already present, and instead does only the build. This is useful when switching between Thin and Full builds.

Note that the Thin/Full choice must match the command line used to build the application.

**macOS build of helloworld:**

```
$ # Thin LTO build
$ ldc2 -flto=thin -L-L./ldc-build-runtime.tmp/lib helloworld.d
```

**Linux build of helloworld:**

```
$ # Full LTO build
$ ldc2 -flto=full -linker=gold -L-L./ldc-build-runtime.tmp/lib helloworld.d
```

The main difference between Linux and macOS is that an alternate (non-default) linker must be specified on Linux.

Any other typical compiler options can be specified as well. For example, a Linux release mode build might be specified as follows:

```
$ # Full LTO release mode build
$ ldc2 -O -release -flto=full -linker=gold -L-L./ldc-build-runtime.tmp/lib helloworld.d
```