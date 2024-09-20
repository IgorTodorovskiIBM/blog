---
layout:       post
title:        "Bash on z/OS: My Journey and Contributions"
author:       "Igor Todorovski"
header-img:   "img/in-post/post-bash/bg.png"
catalog:      true
tags:
    - z/OS
    - Bash
    - Open Source
---

> Bash on z/OS

As part of the [z/OS Open Tools](https://github.com/zopencommunity) initiative, I, along with [Mike Fulton](https://makingdeveloperslivesbetter.wordpress.com), embarked on a journey to port Bash, the renowned shell and scripting language, to the z/OS platform. The motivation was simple: not having Bash on z/OS was a persistant pain point for me and others on z/OS. In this blog, I will discuss the technical challenges encountered while porting Bash to z/OS, and also discuss the steps taken to upstream the z/OS changes back to the Bash community.

![image](https://upload.wikimedia.org/wikipedia/commons/8/82/Gnu-bash-logo.svg)

But first, let's provide a bit of background on why we decided to port Bash:

# The Power of Bash on z/OS:
Bash, short for "Bourne-Again SHell," offers a slew of benefits over the default /bin/sh shell on z/OS. Here are some notable advantages that make Bash an indispensable tool for z/OS users:

* Tab completion: Bash's tab completion feature enhances productivity by allowing users to swiftly complete commands or filenames by typing a few characters followed by the Tab key. This reduces typing errors and accelerates command-line operations.

* Advanced scripting capabilities: Bash serves as a robust scripting language, allowing users to automate complex tasks. Leveraging variables, loops, conditions, and other programming constructs, Bash scripts enable complex automation scenarios.

* Compatibility with the Posix shell: Bash ensures seamless compatibility with the Posix bourne shell. This compatibility simplifies the transition process for z/OS users seeking to harness the power of Bash.

# Porting Bash to z/OS:
Contrary to my initial expectations, porting Bash to z/OS entailed fewer challenges than anticipated, thanks to the awesome efforts by various z/OS teams:

* [Clang compiler availability](https://www.ibm.com/docs/en/open-xl-c-cpp-zos/1.1?topic=new-llvm-clang-infrastructure) on z/OS
  * The latest Clang compiler on z/OS offers better compatibility with other platforms, both from a C and C++ standards perspective and also from an options compatilibity perspective. It also features an ASCII mode option, saving us the hassle of having to transform the open source code to be EBCDIC compatible.
* The [ZOSLIB library](https://github.com/ibmruntimes/zoslib)
  * The ZOSLIB library aims to bridge the gap between the z/OS C LE runtime and the Linux Glibc runtime, and also hides the complexities of dealing with files in different codepages. Remember, z/OS is primarily an EBCDIC system. In summary, ZOSLIB allows us to focus on the application specific issues.

## Notable challanges and resolutions
Although the porting of Bash to z/OS involved minimal changes, we encountered a few issues that required investigation and resolution.

### Supporting external link commands
z/OS includes several commands that serve as external links to z/OS dataset modules. Initially, Bash did not support this functionality. Consequently, when executing the following command in Bash on z/OS, we would get the following error message:

```bash
$ ping www.ibm.com
bash: ping: command not found
```
As seen here, netstat is an external link:
```
erwxrwxrwx   1 ROOT     1              8 Nov 11  2021 ping -> OPING
```
We addressed this issue in this [PR](https://github.com/zopencommunity/bashport/pull/53)

```bash
$ ping www.ibm.com
CS V2R5: Pinging host www.ibm.com (104.70.245.61)
Ping #1 response took 0.018 seconds. (18.276 milliseconds)
```

### Fixing process substitution in z/OS
Another challenge we encountered was related to process substitution on z/OS. 
Process substitution is a useful feature that allows you to use the output of a command or a process as if it were a file. It provides a way to pass the output of a command as an input to another command or perform operations that require a file input.

Unfortuantely, this was broken in our initial port as seen here:

```bash
cat <(date)
(garbled output)
```

After applying the fix in [PR](https://github.com/zopencommunity/bashport/pull/60/files), it worked:
```bash
cat <(date)
Fri May 24 11:34:28 CDT 2023
```

### Fixing the test cases
A significant portion of the effort was dedicated to resolving issues within the test cases. Fortunately, many tests failed due to a common reason: the error messages on z/OS included a prefix that differed from other platforms.

For example, on z/OS, an error message appeared as follows:

```bash
EDC5129I No such file or directory.
```

In contrast, on other platforms, the error message appeared without a prefix:

```bash
No such file or directory
```

Initially, we contemplated changing the expected ".right" files to accommodate the prefix differences. However, after engaging with the upstream maintainer, we made a collective decision to develop a "diff" wrapper that would eliminate the prefix codes specifically on z/OS.

# Contributing to the Bash Community:
Contributing the ported version of Bash back to the broader Bash community marked a significant milestone in this endeavor. The contribution process involved several key steps:

## Step 1: Community engagement: 
After some research, I identified the official Bash community, hosted on [Savannah](https://savannah.gnu.org/projects/bash/). I introduced myself and asked about the contribution guidelines and what the best way was to contribute these changes. I decided to use the Bash mailing list to introduce my intent to contribute the z/OS changes back. Here's my initial [email](https://lists.gnu.org/archive/html/bug-bash/2023-05/msg00048.html):
> Hi there,

> I’m looking for advice on the best way to submit a patch to enable support of 
z/OS.

> We have a few patches here which I will be cleaning up for the next few days: 
https://github.com/zopencommunity/bashport/tree/main/patches

> Is it enough to just provide this patch location once they’re cleaned up? I 
confirmed that they apply to the master branch.

> Or is there a formal process for submitting patches?

> Also, on z/OS we have a prefix message id before the error text as in this 
case: 
https://github.com/zopencommunity/bashport/blob/main/patches/PR3/builtins.right.patch

> Is there a preferred approach for how to handle this? Should I create a 
builtins.right.zos?

> Thanks,
Igor


## Step 2: Building the latest development branch of Bash on z/OS
The next step was to integrate and clean up our changes so that we could build the development branch of Bash on z/OS. This was done as part of [this PR](https://github.com/zopencommunity/bashport/pull/62).

## Step 3: Seeking feedback on z/OS changes
After building the development branch of Bash on z/OS, I continued to engage with Chet Ramey, the maintainer of GNU Bash and provided a link to our working [github branch](https://github.com/zopencommunity/bashport/tree/enable_git). He provided great feedback and was able to simplify and adapt many of my changes!

## Step 4: Iterative refinement
Guided by Chet's expert feedback, I refined the patches iteratively to align with the standards and best practices of the Bash community.

## Step 5: Success: The culmination of the upstreaming effort
The culmination of the upstreaming journey was reached when the patches were successfully integrated into the development branch of Bash:
![image](/blog/img/in-post/post-bash/patch1.png)
![image](/blog/img/in-post/post-bash/patch2.png)
Hooray!

# What's remaining?
* Although all of the source changes to enable Bash on z/OS have been upstreamed, the test case changes have yet to be upstreamed. I'm currently working with Chet on a solution that will not impact the other platforms.

# Special Thanks
I would like to thank Mike Fulton for helping with the Bash on z/OS porting effort.
I would also like to thank Gary Grossi for identifying several key issues with Bash on z/OS.
