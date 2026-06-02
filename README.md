# kallisto for Windows: Unofficial Community Build

This repository provides an unofficial Windows build of
[kallisto](https://pachterlab.github.io/kallisto/) 0.52.0.

kallisto is a command-line program for quantifying abundances of transcripts
from RNA-seq data. The upstream project is primarily built for Unix-like
environments. This repository vendors the kallisto 0.52.0 source tree and
applies small MSYS2-UCRT64 compatibility fixes so that `kallisto.exe` can be
built and used on Windows.

These builds are not produced, endorsed, or supported by the upstream kallisto
project. For kallisto itself, see the upstream project:

https://pachterlab.github.io/kallisto/

## Downloading kallisto for Windows

Prebuilt Windows binaries are available from the
[Releases](https://github.com/win-ngs/kallisto-windows-build/releases) page
of this repository.

Download the latest release archive, for example:

```text
kallisto-0.52.0-windows-ucrt64.zip
```

After extracting the archive, you should see:

```text
kallisto-0.52.0-windows-ucrt64/
  kallisto.exe
  *.dll
  LICENSE.md
  THIRD_PARTY_NOTICES.txt
  LICENSES/
```

Keep all DLL files in the same folder as `kallisto.exe`. The DLLs include the
MSYS2-UCRT64 runtime libraries plus HDF5, HTSlib/BAM, and their runtime
dependencies.

## How to Use

This Windows build uses the same command-line options as upstream kallisto. For
detailed usage and interpretation of results, refer to the upstream kallisto
documentation.

1. Download the ZIP file.
2. Extract the ZIP file.
3. Open PowerShell.
4. Move into the extracted folder.
5. Run `kallisto.exe`.

Example:

```powershell
cd C:\Users\you\Downloads\kallisto-0.52.0-windows-ucrt64
.\kallisto.exe version
.\kallisto.exe --help
```

Build an index and quantify paired-end FASTQ files:

```powershell
.\kallisto.exe index `
  -i C:\data\transcripts.idx `
  C:\data\transcripts.fasta.gz

.\kallisto.exe quant `
  -i C:\data\transcripts.idx `
  -o C:\data\kallisto-out `
  C:\data\reads_1.fastq.gz `
  C:\data\reads_2.fastq.gz
```

Do not move only `kallisto.exe` to another folder, because the `.dll` files in
the ZIP are needed for the program to start.

## Source Tree

The patched upstream source tree is kept under `kallisto-0.52.0-patch/` in this
repository. The binary release ZIP does not include that source tree; it only
contains `kallisto.exe`, required DLLs, and documentation.

```text
kallisto-0.52.0-patch/
```

The upstream kallisto README and license are kept inside that directory:

```text
kallisto-0.52.0-patch/README.md
kallisto-0.52.0-patch/license.txt
```

Build outputs such as `build-ucrt64-clean/` and release ZIP files are not meant
to be committed to git. Release ZIP files should be published through GitHub
Releases.

## Building from Source

You do not need to build kallisto yourself if you only want to use the released
Windows binary. This section is for maintainers or users who want to recreate
the build.

Install [MSYS2](https://www.msys2.org/) first. Open the **MSYS2 UCRT64** shell
from the Windows Start menu. Do not run the build commands from the plain MSYS
shell, MINGW64 shell, CLANG64 shell, PowerShell, or Command Prompt.

In the MSYS2 UCRT64 shell, install the required packages:

```sh
pacman -S --needed \
  base-devel \
  mingw-w64-ucrt-x86_64-toolchain \
  mingw-w64-ucrt-x86_64-cmake \
  mingw-w64-ucrt-x86_64-hdf5 \
  mingw-w64-ucrt-x86_64-htslib \
  mingw-w64-ucrt-x86_64-zlib
```

Clone this repository and move into it:

```sh
git clone https://github.com/win-ngs/kallisto-windows-build.git
cd kallisto-windows-build
```

Configure and build kallisto from the same MSYS2 UCRT64 shell:

```sh
cmake -S kallisto-0.52.0-patch \
  -B build-ucrt64-clean \
  -G "Unix Makefiles" \
  -DCMAKE_MAKE_PROGRAM=mingw32-make \
  -DCMAKE_C_COMPILER=gcc \
  -DCMAKE_CXX_COMPILER=g++ \
  -DUSE_HDF5=ON \
  -DUSE_BAM=ON \
  -DBUILD_FUNCTESTING=ON \
  -DENABLE_AVX2=OFF \
  -DCOMPILATION_ARCH=OFF \
  -DCMAKE_BUILD_TYPE=Release

cmake --build build-ucrt64-clean
```

The executable is created as:

```text
build-ucrt64-clean/src/kallisto.exe
```

## Relationship to Upstream Release Variants

Upstream kallisto 0.52.0 Linux releases provide four tarballs:

```text
kallisto_linux-v0.52.0.tar.gz
kallisto_linux_LongKmer-v0.52.0.tar.gz
kallisto_linux_NoOpt-v0.52.0.tar.gz
kallisto_linux_LongKmer_NoOpt-v0.52.0.tar.gz
```

These names match the four build modes in upstream `.make_binaries.sh`.
In that script, all four modes are configured with:

```text
-DUSE_HDF5=ON -DUSE_BAM=ON -DBUILD_FUNCTESTING=ON
```

The four upstream-style variants differ as follows:

| Variant | Extra CMake options | Practical effect |
|---|---|---|
| default | none | default k-mer limit, Bifrost native CPU tuning and AVX2 enabled by default |
| `LongKmer` | `-DMAX_KMER_SIZE=64` | raises the actual maximum k-mer size from 31 to 63 |
| `NoOpt` | `-DENABLE_AVX2=OFF -DCOMPILATION_ARCH=OFF` | disables AVX2 and `-march=native` CPU-specific tuning |
| `LongKmer_NoOpt` | `-DMAX_KMER_SIZE=64 -DENABLE_AVX2=OFF -DCOMPILATION_ARCH=OFF` | combines long k-mer support with portable CPU options |

This Windows UCRT64 package is currently a portable default-kmer build:

```text
-DUSE_HDF5=ON
-DUSE_BAM=ON
-DENABLE_AVX2=OFF
-DCOMPILATION_ARCH=OFF
```

`MAX_KMER_SIZE` is left at the upstream default of `32`, so the actual maximum
k-mer size is 31. This makes the Windows package closest to the upstream
`NoOpt` variant: HDF5 and BAM are enabled, while AVX2 and `-march=native` are
disabled for CPU portability. It is not the `LongKmer` variant.

`BUILD_FUNCTESTING=ON` does not change `kallisto.exe` features. It only adds
the upstream functional test script as a build target named `test`.

Providing Windows artifacts equivalent to all four official Linux variants
would require separate default, `LongKmer`, `NoOpt`, and `LongKmer_NoOpt`
Windows builds, with the corresponding runtime DLLs and license notices
included for each artifact.

## Validation Performed

This patched build was checked with MSYS2-UCRT64 using:

```text
g++ 16.1.0
CMake 4.3.2
zlib 1.3.2
```

The following checks were run:

```text
cmake configure
build completed
kallisto.exe version
kallisto.exe index with Windows-style C:\... input/output paths
kallisto.exe quant with Windows-style C:\... paths and a trailing backslash output directory
HDF5 output: quant -b 1 produced abundance.h5
HDF5 readback: h5dump converted abundance.h5 back to plaintext
BAM output: quant --pseudobam produced pseudoalignments.bam
upstream test/ data: index + paired-end quant
upstream BUILD_FUNCTESTING test target: index section and quant section
```

The release build disables AVX2 and native CPU tuning so the binary is not tied
to the CPU features of the build machine.

The upstream functional test script writes and hashes text files. Because the
patched build opens plain-text output files in binary mode, `kallisto.exe`
writes LF line endings even on native Windows/UCRT64, so its output is
byte-identical to the upstream Linux output. The **unmodified** upstream test
script was run, with no CRLF-to-LF normalization, and all upstream functional
`index` tests and all `quant` tests passed against the upstream LF-based md5
sums. The script stops before the `bus` section because `bustools` is not
installed in this environment.

`kallisto.exe version` reported:

```text
kallisto, version 0.52.0
```

## MSYS2-UCRT64 Compatibility Patch

The upstream kallisto 0.52.0 source did not build unchanged in this
MSYS2-UCRT64 environment. The compatibility changes are limited to build-system
integration, line-ending handling, and vendored Bifrost source issues exposed by
GCC 16 and the native Windows runtime. Paths below are relative to the upstream
source directory `kallisto-0.52.0-patch/`.

| File | Change | Reason |
|---|---|---|
| `CMakeLists.txt` | Changed `cmake_minimum_required(VERSION 3.0.0)` to `VERSION 3.10` | MSYS2-UCRT64 ships CMake 4.x, which rejects very old policy compatibility |
| `CMakeLists.txt` | Added MinGW defaults: `ENABLE_AVX2=OFF` and `COMPILATION_ARCH=OFF` when those options are not explicitly set | Avoids release binaries that require AVX2 or the build machine's `-march=native` CPU features |
| `CMakeLists.txt` | Replaced Bifrost's in-source `mkdir -p build && cd build && cmake ..` / `make` ExternalProject commands with `${CMAKE_COMMAND} -S/-B`, `BINARY_DIR`, and `cmake --build` | Avoids POSIX shell assumptions and keeps Bifrost out of the source tree |
| `CMakeLists.txt` | Passes the parent generator, `CMAKE_MAKE_PROGRAM`, `CMAKE_C_COMPILER`, and `CMAKE_CXX_COMPILER` into Bifrost | Prevents the parent kallisto build and the Bifrost sub-build from accidentally using different MSYS2 toolchains |
| `CMakeLists.txt` | Moves Bifrost ExternalProject `PREFIX` to `${PROJECT_BINARY_DIR}/bifrost-prefix` and includes both `${binary_dir}/src` and `${source_dir}/src` | Prevents stale shared ExternalProject stamps when using multiple build directories, while still exposing Bifrost generated files and source headers |
| `CMakeLists.txt` | Passes UCRT64 zlib include/library paths into Bifrost when building with MinGW | Prevents Bifrost from finding MSYS zlib while kallisto uses UCRT64 zlib |
| `CMakeLists.txt` | Finds HTSlib from the active UCRT64 compiler prefix when `USE_BAM=ON` | The vendored `ext/htslib` ExternalProject does not build `libhts.a` on Windows |
| `CMakeLists.txt` | Finds HDF5 from the active UCRT64 compiler prefix when `USE_HDF5=ON` | Avoids mixing MSYS HDF5 headers/libraries with UCRT64 builds |
| `src/CMakeLists.txt` | Added `add_dependencies(kallisto_core bifrost)` and `add_dependencies(kallisto bifrost)` | Ensures Bifrost headers and `libbifrost.a` exist before kallisto objects and executable are built |
| `src/CMakeLists.txt` | Links kallisto against `${binary_dir}/src/libbifrost.a` instead of the old in-source Bifrost build path | Matches the out-of-source Bifrost build directory |
| `src/CMakeLists.txt` | Sets `ZLIB_INCLUDE_DIR`, `ZLIB_LIBRARY`, and `ZLIB_LIBRARY_RELEASE` from the active MinGW compiler prefix before `find_package(ZLIB)` | Avoids mixing MSYS headers with UCRT64 headers |
| `src/CMakeLists.txt` | Uses the top-level `HTSLIB_*` and `HDF5_*` paths for include directories and link libraries | Allows `USE_BAM=ON` and `USE_HDF5=ON` to use MSYS2-UCRT64 packages |
| `src/kseq.h` | Changed the local `kstring_t` typedef to use the `struct kstring_t` tag | Matches newer HTSlib headers that forward-declare `struct kstring_t` |
| `ext/bifrost/src/DataStorage.tcc` | Changed `o.sz_link[i].load()` to `o.unitig_cs_link[i].load()` in the copy path | GCC 16 instantiates this template path and catches the invalid member reference |
| `ext/bifrost/src/kseq.h` | Changed Bifrost's local `kstring_t` typedef to use the `struct kstring_t` tag | Prevents a second `kstring_t` conflict when Bifrost and HTSlib headers are included together |
| `ext/bifrost/src/roaring.h` | Adds C++-only `static` linkage to the inline `roaring_bitmap_contains` helper | Avoids a link-time multiple definition with the C implementation in `roaring.c` |
| `ext/bifrost/src/TinyBitmap.cpp` | Replaced `free()` calls for `posix_memalign()`-style TinyBitmap buffers with `posix_memalign_free()` | UCRT64 uses the native Windows CRT, where freeing `_aligned_malloc` memory with `free()` corrupts the heap during `kallisto index` |
| `src/PlaintextWriter.cpp`, `src/PlaintextWriter.h`, `src/main.cpp`, `src/ProcessReads.cpp`, `src/ProcessReads.h`, `src/EMAlgorithm.h`, `src/Inspect.h` | Open plain-text output files with `std::ios::binary` | Forces LF line endings on native Windows so text outputs (`abundance.tsv`, `run_info.json`, `*.mtx`, `transcripts.txt`, `matrix.ec`, etc.) stay byte-identical to upstream Linux output; input parsing is left untouched |

The release executable is a single-dispatch command-line tool. No runtime
dispatcher, manual `PATH` splitting, or `argv[0]`-based companion executable
lookup was found in the kallisto/Bifrost code used by this build. Output path
generation uses direct file opens; Windows-style input and output paths were
checked with `index` and `quant`.

The modified source locations include comments explaining the Windows/UCRT64
change.

## License

kallisto is distributed under the BSD 2-Clause License. See
[LICENSE.md](LICENSE.md). In the repository source tree, the original upstream
license is also kept at `kallisto-0.52.0-patch/license.txt`.

Runtime DLLs included in release ZIP files come from MSYS2 packages and retain
their respective upstream licenses. See
[THIRD_PARTY_NOTICES.txt](THIRD_PARTY_NOTICES.txt) and the [LICENSES](LICENSES)
directory for package and license details.
