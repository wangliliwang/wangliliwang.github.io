---
layout: post
title:  "Golang Memory management: several implements of allocator"
date:   2023-02-08 20:33:49 +0800
categories: Golang 
---

## Introduction 

purpose: indroduce to memory allocation in program runtime.

### vm model in process

1. vm includes stack and heap
2. compiler do escape analyze to decide an object to stack or heap
3. dynamic memory allocators only manage the heap memory

## Basics of memory allocator

### types of allocators 

1. explicit allocator: application allocates and frees space. E.g., `malloc` and `free` in C.
2. implicit allocator: application allocates, but does not free space. E.g., garbage collection in Java, Golang.

### performance goals

1. Throughput: number of completed requests(malloc/free) per unit time. Bigger is better.
2. Peak memory utilization: maximum of `P/H`, `P` is the payload size of objects, `H` is size of heap. Bigger is better.
3. Fragmentation: includes internal and external.

1. graph internal
2. graph external

### methods of keeping track of free blocks

1. implicit free lists
2. explicit free lists
3. segregated free lists

todo several graphs.

## ptmalloc

## tcmalloc 

## jemalloc

## Conclusion

## References

[1] [Dynamic Memory Allocation Lecture 1](http://www.cs.cmu.edu/afs/cs/academic/class/15213-s11/www/lectures/19-allocation-basic.pdf)

[2] [Dynamic Memory Allocation Lecture 2](http://www.cs.cmu.edu/afs/cs/academic/class/15213-s11/www/lectures/20-allocation-advanced.pdf)

[3] [ptmalloc、tcmalloc与jemalloc对比分析](https://www.cyningsun.com/07-07-2018/memory-allocator-contrasts.html)
