---
layout:       post
title:        "Exploring ZOSLIB: An open source z/OS library for porting"
author:       "Igor Todorovski"
header-img:   "img/in-post/ai_on_z.jpg"
catalog:      true
hidden: true
tags:
    - zoslib
    - z/OS
    - Library
    - Porting
---

As an active contributor to the [z/OS Open Tools](https://github.com/ZOSOpenTools) Github organization, I specialize in porting open-source tools to z/OS. Many of the tools the z/OS Open Tools organization ports are written in C or C++. Porting C/C++ software to z/OS can be challenging due to the various differences between Linux systems and z/OS, including C runtime differences ([GNU C Library (Glibc)](https://www.gnu.org/software/libc/) vs [z/OS C Runtime Library](https://www.ibm.com/docs/en/zos/3.1.0?topic=cc-zos-runtime-library-reference)), file system differences (z/OS UNIX/datasets on z/OS), security differences (RACF/SAF on z/OS), and codepages differences (EBCDIC/ASCII program support in z/OS).

As the number of applications and libraries we ported to z/OS increased, we continued to face a recurring issue — essential C runtime APIs were missing and creating custom workarounds at the application level didn't seem like the best solution.

Complicating things further, z/OS UNIX supports file tags that describe a file's encoding content. This file tag metadata doesn't exist in Linux, but it's crucial in z/OS because programs can run in either EBCDIC or ASCII mode. What makes file tags interesting is that, combined with the z/OS Enhanced ASCII services, z/OS can perform automatic conversion of files to and from the program's encoding upon read and write. This enables EBCDIC and ASCII programs to interpret file data correctly, irregardless of the codepage. This feature greatly simplifies handling various codepages in applications. However, although file tags are great, files can also have no tags. How does an application handle such "untagged" files?

That's where ZOSLIB jumps in — it addresses many of these complexities.

## What is ZOSLIB?

At its core, [ZOSLIB](https://github.com/ibmruntimes/zoslib) is an open source extension of the z/OS LE C Runtime Library, offering additional functionalities:

* **Extended POSIX Support**: ZOSLIB includes a subset of POSIX APIs not present in the LE C Runtime Library, addressing critical functionality gaps.
* **Enhanced File Operations**: Improved implementations for C open, pipe, and more which facilitate seamless integration with z/OS auto-conversion facilities.
* **Data Conversion Utilities**: C APIs for EBCDIC to ASCII conversions and vice versa simplify the handling of different character encodings.
* **Diagnostic Reporting**: ZOSLIB provides APIs for enhanced diagnostic reporting, aiding in troubleshooting.

ZOSLIB was originally introduced as a library to aid in porting Node.js and its components to z/OS. It has since been open-sourced and is now used by over 100 open-source projects under the z/OS Open Tools organization.

The ZOSLIB project is available on GitHub at [https://github.com/ibmruntimes/zoslib](https://github.com/ibmruntimes/zoslib). It is licensed under the Apache 2.0 license, which imposes few restrictions and facilitates modification and redistribution.

ZOSLIB is a 64-bit ASCII library and is meant to be leveraged by ASCII-based applications and libraries.

Now, let's explore the gaps that ZOSLIB aims to fill.

## Gap 1: How do we handle untagged files?

File tags are very useful metadata for determining the underlying codepage of a file. However, z/OS does not make this metadata field mandatory. z/OS UNIX files can lack a tag, and such files are considered untagged. Complicating matters further is that by default, C LE `open()` and `creat()` functions generate untagged files. Additional code is required to tag newly created files. This is especially problematic when porting open source code that has no notion of file tags. 

So why are untagged files an issue? This creates a problem because the application can not determine the true encoding of untagged files. This means that an EBCDIC-based program expecting text data will interpret the data one way and an ASCII-based program will interpret the data another way.

To address this issue, in zoslib, we implemented an override for the C `open()` function that incorporates a heuristic (toggleable if needed by an environment variable), to determine the true encoding of untagged files. Additionally, since z/OS creates untagged files (and pipes) by default, and there is no toggle to control this behavior, we decided to override C `open()`, 'pipe()','creat()' to additionally tag all files to ASCII ISO8859-1 by default. This override can be adjusted via an environment variable (documented in the zoslib manpages). Currently, this feature only works for z/OS UNIX files but could potentially be extended to cover datasets as well. 

When zoslib is added to your project and the macro -DZOSLIB_OVERRIDE_CLIB=1 is set, all references to `open` are mapped to a new zoslib `__open_ascii` function (defined [here](https://github.com/ibmruntimes/zoslib/blob/3a2b4b06aa52095be6c805f921b101e9ff8c9a15/src/zos-io.cc#L808)).

This enhancement has enabled zoslib applications to seamlessly work with both legacy untagged data and tagged data.

## Gap 2: Missing C Runtime APIs

ZOSLIB also enables us to implement missing C runtime functions once and then make them available to all projects leveraging zoslib. For instance, several projects have transitioned to using `posix_spawn` rather than `fork`/`exec`. Rather than implementing `posix_spawn` for each individual application, we can implement it once in zoslib and leverage it across many projects. Additional functions added to zoslib include `mkdtemp`, `execvpe`, `strpcpy`, `strsignal` and many more, all of which are also available in GlibC.

Additionally, there are instances where a header file may be missing. For example, on z/OS, `sys/param.h` does not exist, and the macro definitions that are typically in `sys/param.h` are present in other headers. However, the assumption by open source projects is that this header exists across all operating systems. Without guarding out the header include, the compiler would trigger an error. However, this means making an application specific change. By adding a `sys/param.h` header to zoslib, we avoid yet another change to the application's source code.

The good news is that many of these APIs are currently being implemented in the z/OS LE C runtime library, but zoslib enables us to adopt an agile approach to development, allowing us to promptly address these gaps as we encounter them.

## Gap 3: Provide System-Level Interfaces for z/OS

Operating systems like Linux also provide system-level functions, such as `sysconf(_SC_NPROCESSORS_ONLN);`, which retrieves the number of online CPUs for the system. Such system-level functions vary from platform to platform, and offering interfaces to capture equivalent implmentations aids in porting.

For instance, the following functions have been added to zoslib:
* `__get_num_online_cpus` - capture the number of online cpus
* `__get_os_level` - return the operating system level
* `__get_cpu_model` - returns the cpu model

These are just a few examples of what ZOSLIB has to offer and I encourage to read the source and examine the zoslib documentation for more information.

## How we're porting tools with ZOSLIB

Now that we've examined what ZOSLIB has to offer, let's explore how you can seamlessly integrate ZOSLIB into your application or library.

To leverage ZOSLIB, you can either compile it from source as described in the [README](https://github.com/ibmruntimes/zoslib) or you can download a release from the z/OS Open Tools [zoslibport repository](https://github.com/ZOSOpenTools/zoslibport/releases).

This release provides the libraries (static and dynamic), include headers files, man pages, as well as environment variables for use in the zopen framework.

We'll first cover how to integrate zoslib into your code manually, and then describe how we're automatically integrating zoslib into the 100+ z/OS Open Tools projects.

## Integrating zoslib manually

First, download a stable release of zoslib [here](https://github.com/ZOSOpenTools/zoslibport/releases). If you have the z/OS Open Tools package manager, `zopen`, on your system (details [here](https://zosopentools.github.io/meta/#/Guides/QuickStart)), you can simply run:
```
zopen install zoslib
```

Alternatively, you can download the pax.Z asset and extract it onto your system.

Let's assume you have a simple hello world program that writes a file to disk.
```
//a.c
#include <fcntl.h>
#include <unistd.h>

int main() {
    const char *filename = "output.txt";
    const char *message = "Hello, World!\n";

    int file = open(filename, O_WRONLY | O_CREAT | O_TRUNC, 0666);

    if (file == -1) {
        perror("Error opening file");
        return 1; 
    }

    ssize_t bytes_written = write(file, message, strlen(message));

    if (bytes_written == -1) {
        perror("Error writing to file");
        close(file);
        return 1; 
    }

    close(file);

    return 0; 
}
```

The below example uses the clang compiler, available [here](https://epwt-www.mybluemix.net/software/support/trial/cst/programwebsite.wss?siteId=1803). If you do not have clang on your z/OS system, you can use `xlc` or `xlclang`.

Now let's compile this C program using clang as follows:
```
clang -fzos-le-char-mode=ascii -m64 -o a.out a.c
./a.out # run it
ls -lT output.txt # check the tag
```

You will notice that the output.txt file is untagged.

Now let's incorporate zoslib. Set ZOSLIB_HOME to the path to your zoslib installation.
```
clang -fzos-le-char-mode=ascii -m64  -DZOSLIB_OVERRIDE_CLIB=1 -isystem $ZOSLIB_HOME/include -o a.out -lzoslib $ZOSLIB_HOME/lib/celquopt.s.o
./a.out
ls -lT output.txt
```

As you can see, the output.txt file is now tagged as ISO8859-1. This is because `open()` is now overriden with `__open_ascii()` from zoslib as instructed by the macro `-DZOSLIB_OVERRIDE_CLIB=1`/

Now let's examine a simpler way of integrating ZOSLIB, and that is by leveraging the z/OS Open Tools framework. This is how zoslib is currently being leveraged in over 100+ z/OS Open Tools projects.

## Leveraging the z/OS Open Tools Build Framework

Examine the following code snippet:
```
export ZOPEN_BUILD_LINE="STABLE"
export ZOPEN_STABLE_URL="http://www.greenwoodsoftware.com/less/less-${LESS_VERSION}.tar.gz"
export ZOPEN_STABLE_DEPS="make curl gzip tar ncurses zoslib"
```
The above snippet, taken from [lessport](https://github.com/ZOSOpenTools/lessport/blob/main/buildenv) describes how simple it is to leverage zoslib using the zopen build framework.
Simply adding zoslib as a dependency (in `ZOPEN_STABLE_DEPS`) will instruct the build to add zoslib as a dependency and pass in the zoslib compiler flags. These flags automatically instruct the compiler to pick up the necessary header files and to link to the ZOSLIB static archives. If you're wondering, these flags are passed in via the .appenv file from your ZOSLIB_HOME installation. The file is generated by the [zoslibport build configuration](https://github.com/ZOSOpenTools/zoslibport/blob/4bf40195e8c646d34b3fdc6299c0f00a3f9442f1/buildenv#L45).

The `zopen` build framework sets the CPPFLAGS, CXXFLAGS, CFLAGS, LIBS, LDFLAGS accordingly. Fortunately, almost every projects we deal with rely on these flags, meaning that we can easily pass integrate ZOSLIB as an additional library to any port.

Since late 2022, several projects, including Git, Curl, Bash and many more have been successfully integrated with zoslib. Open Source communities have also adopted our dependency on zoslib, including GNU Make, Bash, Vim and more!

If you're interested in porting to z/OS Open Tools and would like to learn how, check out our documentation [here](https://zosopentools.github.io/meta/#/Guides/Porting).

## Contributing to ZOSLIB

Contributions to ZOSLIB are encouraged, and the process involves opening pull requests at https://github.com/ibmruntimes/zoslib/pulls. Following the contribution guidelines ensures a smooth integration of your changes into the z/OS Open Source community.

## Future considerations
* Consider creating routines to expose z/OS services like SAF / CMS APIs
* Adding support for handling datasets seamlessly

# Special Thanks
Thank you to Mike Fulton, Gaby Baghdadi, Wayne Zhang, Sean Perry, Eric Janssen, Haritha D and many others for their contributions to ZOSLIB.
