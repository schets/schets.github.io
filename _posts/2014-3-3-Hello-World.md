---
layout: post
title: Implementing and optimizing a simple slab allocator
---

*Code and post in progress, [code here](github.com/schets/fast_alloc)*

#Intro#

blah blah blah memory is important malloc is slow many datastructures are only accessed by one thread at a time write this later ...

...

...

#Free Lists#
[Free lists](https://en.wikipedia.org/wiki/Free_list) are one of the most convinient methods of creating an allocator, and are pretty easy to implement. They're used in some of the most common allocators used today, including [ptmalloc (glibc malloc)](http://code.woboq.org/userspace/glibc/malloc), [jemalloc](http://www.canonware.com/jemalloc/) and [tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html), as well as in many [specialized allocators](http://gameprogrammingpatterns.com/object-pool.html).

The basic idea behind a free list is that one stores a linked list of open memory chunks that acts as a stack for allocating (popping a chunk) and freeing (pushing a chunk). For our purposes, this data structure will resemble

```C
struct free_list_chunk {
    //potential_metadata here;
    //void *stuff;

    union {
        struct free_list_chunk *next_chunk;
        char data[];
    } chunk;
};
```

The simplest allocator that can be created with a free list works in a fixed block of memory and simply pushes/pops chunks from the list

```C
#define NEXT_CHUNK(x) *(void **)x

union fixed_list_chunk {
    union fixed_list_chunk *next;
    char data[];
};

struct fixed_list {
    union fixed_list_chunk *head; //head is void since is union of uknown types
}

void *fixed_list_malloc(struct fixed_list *from) {
    if(!from->head)
        return NULL;
    void *data = from->head;
    from->head = NEXT_CHUNK(from->head);
    return data;
}

void *fixed_list_free(struct fixed_list *from, void *freeptr) {
    if(!freeptr)
        return NULL;
    NEXT_CHUNK(freeptr) = from->head;
    from->head = freeptr;
}
```

along with the initialization and freeing functions

```C
fixed_list create_fixed_list(size_t object_num, size_t object_size) {
    fixed_list retlist;
    object_size = object_size < sizeof(void *) ? sizeof(void *) : object_size; //also padding...
    retlist.head = (fixed_list_chunk *)malloc(object_num * object_size);
    if(retlist.head) {
        void *cur_chunk = retlist.head;
        for(size_t i = 0; i < object_num - 1; i++) {
            void *next_chunk = (char *)cur_chunk + object_size;
            NEXT_CHUNK(cur_chunk) = next_chunk;
            cur_chunk = next_chunk;
        }
        NEXT_CHUNK(cur_chunk) = NULL;
    }
    return retlist;
}
```
