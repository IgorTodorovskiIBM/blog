---
layout:         post
title:          "My Journey Porting ccache to z/OS for Faster Builds"
author:         "A z/OS Open Source Developer"
header-img:     "img/in-post/ccache.png"
hidden:         true
catalog:        true
tags:
    - ccache
    - performance
    - build
    - z/OS
    - zopen
---

As a developer in the [zopen community](https://zopen.community/) (a z/OS Open Source Community), I've lost what feels like countless hours waiting for builds to finish. The build-time bottleneck becomes especially painful in an automated CI/CD pipeline, where I'm waiting on a critical fix to be built, verified, and deployed. I personally don't enjoy waiting around, nor do our users, so this drove me to find a better way, and my search led me to `ccache`, a tool that caches compiler output to speed up rebuilds. The caching approach had me intrigued, so I decided to port ccache to z/OS and integrate it into the zopen community to see if it can help!

In this blog, we’ll cover the basics of ccache, how to get it running on z/OS, how to use it effectively, and most importantly, how it can cut your build times in half.

### What is `ccache` and Why Should I Care?

`ccache` is a compiler cache. Its function is to speed up recompilation by caching the results of previous compilations. When rebuilding software, `ccache` can provide these cached outputs instead of re-running the compiler, which reduces build time. This should result in a much faster build turnaround-time.

### How `ccache` Works 

The beauty of `ccache` is its simplicity. It acts as a "wrapper" that intercepts calls to your real compiler (in my case, **IBM Open XL C/C++'s clang compiler**). It then quickly determines if it has a valid, cached result (usually a compiled object) from a previous identical compilation.

This diagram demonstrates the workflow:


![image](/blog/img/in-post/ccache.png)


**The Process**

1.  **Analysis and Hashing:** When your build process calls `ccache`, it analyzes the command and runs the C/C++ preprocessor on your source file. This is a key step because `ccache` hashes the preprocessed output, meaning **it's fully aware of header file changes**.
2.  **"Cache Hit":** If `ccache` finds a previously stored object file that matches this exact hash, it then copies the cached result into the right place. This is the fast path.
3.  **"Cache Miss":** If no match is found, `ccache` calls the real compiler to do the work. Upon success, it saves the new object file into its on-disk cache, ready for next time.

For more in-depth details, visit the official `ccache` website: **[https://ccache.dev](https://ccache.dev)**

### Using `ccache` on z/OS: From Simple to Integrated

#### 1. Installing `ccache`

Begin by installing or updating Vim on your system using the [zopen package manager](https://github.com/zopencommunity/meta):

```bash
zopen install ccache 
```

#### Leveraging Ccache

There are several ways to leverage `ccache`. You can use it for a single compilation, integrate it into your own `Makefile`, or use the seamless support within the `zopen` build framework.

#### Simple Usage: Caching a Single File

For a quick test or a one-off compilation, you can simply prefix your compiler command with `ccache`.

```bash
# Instead of: clang -c myprogram.c -o myprogram.o
# Use:
ccache clang -c myprogram.c -o myprogram.o
```
The first time you run this, `ccache` will compile and cache the result. The second time, it will be nearly instant.

#### Integration into Makefiles: Using `ccache` in a Makefile

For any personal or non-zopen project you build with `make`, the most common way to use `ccache` is to set the `CC` variable in your `Makefile`. This tells `make` to use `ccache` for all C compilations.

```makefile
# In your Makefile
CC = ccache clang
CFLAGS = -m64 -O3 -g

my_program: main.o utils.o
	$(CC) $(CFLAGS) -o my_program main.o utils.o

main.o: main.c
	$(CC) $(CFLAGS) -c main.c

utils.o: utils.c
	$(CC) $(CFLAGS) -c utils.c
```
With this setup, every time you run `make`, the compilation steps will automatically benefit from caching.

### Benchmarking `ccache` with the zopen Build Framework

While the manual methods are great, I wanted to test `ccache` in a fully integrated scenario for zopen. The `zopen` project provides a build framework (`zopen build`) that automates the entire build process, and it now has built-in support for `ccache` via a simple `--ccache` flag. I used the **GNU `bash`** port ([bashport](https://github.com/zopencommunity/bashport)) as my test project.

**The Setup**

My first step was to clone the `bashport` repository from the zopen community.
```bash
git clone https://github.com/zopencommunity/bashport.git
cd bashport
```

**The Experiment**

My experiment was designed to mimic a CI/CD workflow where the workspace is cleaned before each run. I performed three builds:
1.  A baseline build without `ccache`.
2.  A build with `--ccache` on an empty cache.
3.  A second build with `--ccache` on a populated cache.

**1. The Baseline Test (Without `ccache`)**

First, I timed a standard, full build using `zopen build` with verbose flags.
```bash
echo "Starting baseline 'zopen build'..."
time zopen build -vv
```
**Baseline Full Build Time:**
* `real    8m43.963s` (approximately **524 seconds**)
* The zopen build framework, which times the core build phase separately from configure and install, reported: `VERBOSE: build completed in 237 seconds.`

This nearly 9-minute full-build time, and more specifically the 237-second build phase, became my baseline.

**2. The `ccache` Test (Cold Cache)**

Next, I removed the build artifacts to simulate a clean workspace and ran the build again, this time adding the `--ccache` flag. Before this step, it's important to have the necessary tools. 

This experiment requires `ccache`, which you can install via `zopen install ccache`. You also need a supported compiler. I used the `clang` compiler provided by **IBM Open XL C/C++**.

```bash
rm -rf bash-* # remove the bash source/build directory
time zopen build -vv --ccache
```

**Cold `ccache` Build Time:**
* `real    9m16.050s` (approximately **556 seconds**)
* The zopen build framework, which times the core build phase separately from configure and install, reported: `VERBOSE: build completed in 251 seconds.`

As my results show, the very first build with `ccache` was slightly *slower* than the baseline. This is expected! It's the one-time cost of `ccache` hashing every single file and populating its cache for the first time. The real magic comes next.

**3. The `ccache` Test (Warm Cache):**

This is the key test. It simulates all subsequent builds in a CI pipeline or a developer's clean rebuild after the cache has been populated. I again removed the build artifacts and re-ran the exact same command.
```bash
rm -rf bash-* # remove the bash source/build directory
echo "Starting 'zopen build --ccache' on a warm cache..."
time zopen build -vv --ccache
```
**Warm `ccache` Build Time:**
* `real    4m2.766s` (approximately **243 seconds**)
* The zopen build framework reported: `VERBOSE: build completed in 12 seconds.`

The results were great! The total time dropped from **8m43s** to **4m2s**. That's a **54% reduction (more than 2x faster)** in the total time I had to wait!

The framework's build timer tells an better story: the core build phase, which was **237 seconds** in the baseline, dropped to just **12 seconds**. This shows that with a warm cache, the time-consuming compilation step was almost entirely eliminated, leaving only the much faster linking step within the build phase. While the configure and install phases took a similar amount of time in each run, the massive speedup in the build phase is what drove this incredible overall improvement.

### Managing and Monitoring Your Cache

Using `ccache` is great, but knowing how to manage is also critical.

* **Checking Statistics:** To see how effective `ccache` is, use the `-s` flag. It provides a human-readable summary of cache hits, misses, cache size, and more.
    ```bash
    ccache -s
    ```
* **Clearing the Cache:** To start completely fresh, you can clear the entire cache.
    ```bash
    ccache -C
    ```
* **Setting Cache Size:** You can control the maximum size of the cache. For example, to set a 5 GB limit:
    ```bash
    ccache -M 5G
    ```

### Future Considerations

While ccache delivers clear performance gains for C/C++ with modern compilers like Clang, traditional z/OS compilers—such as those for COBOL or PL/I aren’t supported. 

This raises important questions:

* Should we explore caching strategies for legacy languages on z/OS?
* What impact could this have on build speed, especially in large enterprise pipelines?

