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

The basic idea behind a free list is that one stores a linked list of open memory chunks that acts as a stack for allocating and freeing. For our purposes, the memory chunk layout will resemble

```C
struct free_list_chunk {
    //potential_metadata here;
    //void *stuff;

    union {
        struct free_list_chunk *next_chunk;
        char data[alloc_size];
    } chunk;
};
```

The simplest allocator that can be created with a free list is

```C
union fixed_list_chunk {
    union fixed_list_chunk *next;
    char data[1]; //needs a size, char c[] not supported in unions
};

struct fixed_list {
    union fixed_list_chunk *head;
    void *data;
}

void *fixed_list_malloc(struct fixed_list *from) {
    if(!from->head)
        return NULL;
    void *data = from->head->data;
    from->head = from->head->next;
    return data;
}

void fixed_list_free(struct fixed_list *from, void *freeptr) {
    if(!freeptr)
        return NULL;
    ((fixed_list_chunk *)freeptr)->next = from->head;
    from->head = freeptr;
}
```

fixed_list_malloc simply pops a chunk of memory of the head of the list while fixed_list_free pushes the chunk on.

All that's needed are initialization and deletion methods

```C

#define ALIGN_TO sizeof(void *)
//Could be 16 bytes, or an allocator parameter

size_t pad_size(size_t start_size) {
    //If your compiler supports variable length arrays, this could be
    //sizeof(union {void *stuff, char[start_size] data;});
    
    if(start_size < ALIGN_TO)
        return ALIGN_TO;
    else if (!(start_size % ALIGN_TO))
        return start_size;
    return start_size + (ALIGN_TO - (start_size % ALIGN_TO))
}

struct fixed_list create_fixed_list(size_t object_num, size_t object_size) {
    fixed_list retlist;

    //pad the data to ensure proper alignment on each allocation
    object_size = pad_size(object_size);

    retlist.data = malloc(object_num * object_size);
    retlist.head = retlist.data;

    if(retlist.data) {
        union fixed_list_chunk *cur_chunk = retlist.data;
        for(size_t i = 0; i < object_num - 1; i++) {
            void *cur_chunk->next = (char *)cur_chunk + object_size;
            cur_chunk = cur_chunk->next;
        }
        cur_chunk->next  = NULL;
    }
    return retlist;
}

struct fixed_list destroy_fixed_list(fixed_list *inlist) {
    free(retlist.data);
}

```

And we have a working (albeit rudimentary) allocator
