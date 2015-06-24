---
layout: post
title: Implementing and testing a simple slab allocator
---

*Code and post in progress, [code here](github.com/schets/fast_alloc)*

#Intro#

blah blah blah memory is important malloc is slow many datastructures are only accessed by one thread at a time write this later ...

[jemalloc](https://www.facebook.com/notes/facebook-engineering/scalable-memory-allocation-using-jemalloc/480222803919)

[Object Pool](http://gameprogrammingpatterns.com/object-pool.html)

[glibc malloc](http://code.woboq.org/userspace/glibc/malloc)

...

...

#Free Lists#
[Free lists](https://en.wikipedia.org/wiki/Free_list) are one of the most convinient methods of creating an allocator, and are pretty easy to implement. They're used in some of the most common allocators used today, including [ptmalloc (glibc malloc)](http://code.woboq.org/userspace/glibc/malloc), [jemalloc](http://www.canonware.com/jemalloc/) and [tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html), as well as in many [specialized allocators](http://gameprogrammingpatterns.com/object-pool.html). The basic idea is that one stores a linked list of free memory chunks that acts as a stack for allocating (popping a chunk) and freeing (pushing a chunk). For our purposes, this data structure will resemble
```C

struct free_list_chunk {
    //allocator metadata here
    //size_t alloc_size;
    //void *stuff;

    union {
        struct free_list_chunk *next_chunk;
        char data[alloc_size];
    } chunk;
};

```
