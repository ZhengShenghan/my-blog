---
title: "How to Run Symbolic Execution on Darpa CGC"
summary: Customization of Darpa CGC code to run symbolic execution
date: "2025-05-01"
weight: 1
aliases: ["/papermod-darpaklee"]
tags: ["Symbolic Execution", "Darpa CGC", "Program Analysis"]
author: ["Anonymous Bear"]
social:
  github: "ZhengShenghan"
ShowToc: true
TocOpen: true
---

### Introduction

- The DARPA Cyber Grand Challenge (CGC) presented a unique set of challenges for automated vulnerability discovery. One of the key limitations in analyzing CGC binaries with tools like symbolic execution was the specific nature of the code and the environment it ran in.

- In this blog post, we'll delve into the hurdles encountered when attempting to apply symbolic execution to CGC code and explore potential modifications to make these binaries more amenable to this powerful analysis technique. We'll discuss the architectural differences, specific code patterns, and environmental factors that posed difficulties.

- This post aims to be a starting point for researchers and practitioners interested in pushing the boundaries of automated binary analysis on complex, competition-style code.

---

### Background

- Klee is a dynamic symbolic execution engine built on the LLVM compiler infrastructure. It takes bitcode files as input and executes them symbolically to traverse different paths and find vulnerabilities.
---


### Challenges in Applying Symbolic Execution to CGC Binaries

- Running KLEE on Darpa's CGC program has been a problem for a long time. @moyix proposed a [solution](https://gist.github.com/moyix/71357ca3b4d65c13e8394517e7093e21) to make KLEE runnable in CGC programs.
To summarize, applying KLEE on CGC programs have the following obstacles:

Non-standard main function: CGC binaries often don't use the standard int main(int argc, char *argv[]), sometimes having int main(void) or even using the first argument for the flag page address (though the latter is reportedly fixed in a development branch).


Symbol clashes with libc: Many CGC binaries define symbols that conflict with standard C library symbols (like stdin), causing issues when linking with KLEE's uclibc.


Difficult bitcode extraction: Obtaining LLVM bitcode, which KLEE requires, from the CGC binaries is not straightforward and currently involves using wllvm to build and then extract-bc.


32-bit vs. 64-bit architecture mismatch: CGC binaries are 32-bit, while most development systems are 64-bit. Cross-compiling KLEE and its dependencies for 32-bit is complex and can lead to issues like malloc() returning addresses too large for 32-bit pointers (a problem with a known workaround and potential fix in a newer KLEE allocator).


Handling of the "flag page": CGC binaries use a "flag page" (random data at a fixed memory address) implemented with mmap, which KLEE doesn't model.


Use of mmap in allocate: The CGC's allocate function also uses mmap. A workaround involves changing it to use malloc() and mprotect() (though KLEE doesn't fully understand mprotect).


- During my exploration, I encountered some similar problems and found this post. However, it looks like it still need some refinement to run KLEE and this inspires me to write this blog.

---

### Steps to run Symbolic Execution in Darpa CGC

- We will use docker to facilitate the process and improve reproducibility. I use Ubuntu 20.04 to run the docker. We use wllvm to compile to code  

To pull a Klee image with a specific tag:
```
$ docker pull klee/klee:3.0
```
For the CGC challenges, Tag 3.0 is recommended since it has llvm-13 and
wllvm installed, which are needed to use with Klee.

Then, we create and run the Docker container using this Klee image:
```
$ docker run-ti--name=klee3.0_container--ulimit=’stack=-1:-1’ klee/klee:3.0
```
Inside the container, set up the following environment variables, needed for
compilation:
```
$ export LLVM_COMPILER=clang 
$ export LLVM_COMPILER=clang
$ export LLVM_COMPILER=clang
```
For the Darpa CGC program, we will use @hengyin's repo.
```
$ git clone https://github.com/hengyin/cb-multios.git
```
We make the following change of build.sh to avoid compilation errors:

Comment out line 8-15
```
#if [[ -z "${NO_PYTHON_I_KNOW_WHAT_I_AM_DOING_I_SWEAR}" ]]; then
  # Install necessary python packages
#  if ! /usr/bin/env python -c "import xlsxwriter; import Crypto" 2>/dev/null; then
#      echo "Please install required python packages" >&2
#      echo "  $ sudo pip install xlsxwriter pycrypto" >&2
#      exit 1
#  fi
#fi
```
Change CC/CXX variable on line 25 and 26
```
CC=${CC:-wllvm}
CXX=${CXX:-wllvm++}
```
Edit the following files as suggested by @moyix:

cb-multios/include/libcgc.c
```
line 158
void *return_address = malloc(length);
line 160-163
if (return_address == NULL) {
  return errno;
}
mprotect(return_address, length, page_perms);
line 229-238
static void __attribute__ ((constructor)) cgc_initialize_flag_page(void) {
#if 0
  void *mmap_addr = mmap(CGC_FLAG_PAGE_ADDRESS, PAGE_SIZE,
                         PROT_READ | PROT_WRITE,
                         MAP_FIXED | MAP_PRIVATE | MAP_ANONYMOUS,
                         -1, 0);

  if (mmap_addr != CGC_FLAG_PAGE_ADDRESS) {
    err(1, "[!] Failed to map the flag page");
  }
#endif
  void *mmap_addr = malloc(PAGE_SIZE);  
```
cb-multios/CMakeLists.txt
```
line 56-61
    add_compile_options(
        -fno-builtin
        -w
        #-g3
        #-m32
        -g
        -o0 -Xclang -disable-o0-optnone
    )
```

Run script, this will compile all of the challenges within the cb-multios/challenges
folder. 
```
$ ./build.sh
```
extract bitcode
```
$ extract-bc <challenge_executable>
```
---