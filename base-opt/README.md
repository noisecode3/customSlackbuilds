# base-opt

A collection of SlackBuild scripts that installs a self-contained build
environment under `/opt/base`, based on packages from Slackware current.

The purpose is to provide a newer glibc, toolchain, and supporting libraries
on a Slackware 15.0 system without touching or replacing any system files.
All packages in this collection install into `/opt/base` with a unified
directory layout: `/opt/base/bin`, `/opt/base/lib`, `/opt/base/lib64`,
and so on, just like a normal root filesystem but rooted at `/opt/base`,
without the usr split and kept small.

Other packages that require a newer glibc or toolchain should list
`base-opt` as a dependency and link against `/opt/base`.


## Why this exists

Slackware 15.0 ships glibc 2.33. Some software requires a newer glibc at
build time or link time. You cannot simply drop a new `libc.so.6` next to
the system one — `ld-linux` and `libc` are a matched pair and must be
completely isolated from each other. Mixing versions crashes the dynamic
linker before anything else can run.

The solution is a self-contained environment in `/opt/base` where the new
`ld-linux-x86-64.so.2`, `libc.so.6`, compiler, and tools all see only each
other. Binaries built against base-opt have the new dynamic linker path
burned into their ELF header via `-Wl,--dynamic-linker` and find their
libraries via an rpath pointing into `/opt/base`. They run alongside normal
Slackware binaries without conflict.


## Packages

The collection is split into separate SlackBuilds so individual components
can be rebuilt or updated independently. All of them install under the same
`/opt/base` prefix. Build and install them in the order listed below.
Notice how we rebuild gcc after glibc.

### bootstrapping order

| Package              | Stage                                                    |
|----------------------|----------------------------------------------------------|
| `base-opt-binutils`  | Against system glibc, needed to build new glibc (Stage 1)|
| `base-opt-gcc`       | Against system glibc, needed to build new glibc (Stage 1)|
| `base-opt-glibc`     | Build once (Stageless)                                   |
| `base-opt-zlib`      | Build once (Stageless)                                   |
| `base-opt-xz  `      | Build once (Stageless)                                   |
| `base-opt-lz4  `     | Build once (Stageless)                                   |
| `base-opt-zstd`      | Build once (Stageless)                                   |
| `base-opt-jansson`   | Build once (Stageless)                                   |
| `base-opt-bzip2`     | Build once (Stageless)                                   |
| `base-opt-elfutils`  | Build once (Stageless)                                   |
| `base-opt-libfl`     | Build once (Stageless)                                   |
| `base-opt-binutils`  | Rebuild against zlib zstd jansson (Stage 2)              |
| `base-opt-gmp`       | Build once (Stageless)                                   |
| `base-opt-mpfr`      | Build once (Stageless)                                   |
| `base-opt-mpc`       | Build once (Stageless)                                   |
| `base-opt-isl`       | Build once (Stageless)                                   |
| `base-opt-gcc`       | Rebuild against binutils zlib zstd (Stage 2)             |

`base-opt-glibc` has no dependencies inside base-opt and must be installed
 as the first library. Everything else depends on it and `base-opt-gcc` must 
 come before and after glibc. First because glibc needs newer API/ABI. 
 Then those needs to be rebuild against glibc.

## Directory layout

```
/opt/base/
  bin/
  etc/
  include/
  lib/
    pkgconfig/
  lib64/
    pkgconfig/
  sbin/
  share/
  var/
```

No files are placed outside `/opt/base`. The host system is not modified.

## Building a package against base-opt

In your SlackBuild, export these before running configure:

```bash
export PATH=/opt/base/bin:$PATH
export CC="/opt/base/bin/gcc"
export CXX="/opt/base/bin/g++"
export PKG_CONFIG_PATH="/opt/base/lib64/pkgconfig"
export CFLAGS="-I/opt/base/include $SLKCFLAGS"
export CXXFLAGS="-I/opt/base/include $SLKCFLAGS"
```

Then invoke configure and make via the new dynamic linker so the build
tools themselves run against the new glibc:

Verify the finished binary before packaging:

```bash
# Should resolve libraries under /opt/base exexpt if you installe others in /opt or /home
# But they should not mix with /usr/lib{64,} /lib{64,}
# Not if you did not run export PATH=/opt/base/bin:$PATH it will report wrong glibc
ldd ./your-binary
./your-binary: /lib64/ld-linux-x86-64.so.2: version `GLIBC_2.35' not found (required by /opt/base/lib64/libc.so.6)
	linux-vdso.so.1 (0x00007fbde6e9e000)
	libc.so.6 => /opt/base/lib64/libc.so.6 (0x00007fbde6a00000)
	/opt/base/lib64/ld-linux-x86-64.so.2 => /lib64/ld-linux-x86-64.so.2 (0x00007fbde6ea0000)

# Should show /opt/base/lib/ld-linux-x86-64.so.2
readelf -l ./your-binary | grep interpreter
      [Requesting program interpreter: /opt/base/lib64/ld-linux-x86-64.so.2]

readelf -d ./your-binary | grep -i 'rpath\|runpath\|needed'
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000f (RPATH)              Library rpath: [/opt/base/lib64:/opt/base/lib]

```

## Notes

Slackware current packages versions are used as the upstream source but modified to
be as minimal as possible. Sources are fetched directly via git which keeps 
the SlackBuilds simple to just change shasums and version numbers. To avoid the
bootstrapping problem — a new glibc cannot be compiled by the old toolchain, 
so we repackage from current instead of building from scratch.

The dynamic linker path `/opt/base/lib/ld-linux-x86-64.so.2` is burned
into every binary built against base-opt. These binaries are fully
self-contained with respect to glibc and will continue to work even if
base-opt is later updated, as long as `/opt/base/lib` remains in place,
but can be installed anywhere even in just in you're home folder.
