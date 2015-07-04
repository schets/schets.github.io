---
layout: post
title: Free lists and per-datastructure allocators
---

*Code and post in progress, [code here](https://github.com/schets/fast_alloc)*

#Intro#

blah blah blah memory is important malloc is slow many datastructures are only accessed by one thread at a time cache locality write this later ...

...

...

#Free Lists#
[Free lists](https://en.wikipedia.org/wiki/Free_list) are one of the most convinient methods of creating an allocator, and are pretty easy to implement. They're used in some of the most common allocators used today, including [ptmalloc (glibc malloc)](http://code.woboq.org/userspace/glibc/malloc), [jemalloc](http://www.canonware.com/jemalloc/) and [tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html), as well as in many [specialized allocators](http://gameprogrammingpatterns.com/object-pool.html).

The idea behind a free list is that one stores a linked list of open memory chunks that are used to satisfy allocation requests. An element of a free list will usually resemble somthing like

```C
struct free_list_chunk {
    //persistent metadata here;
    //void *stuff;

    union {
        struct {
            struct free_list_chunk *next_chunk;
            struct free_list_chunk *prev_chunk;
            //possibly other metadata here
        } epheremal_metadata;
        char chunk[alloc_size];
    } chunk;
};
```
Each chunk consists of three elements:

- Persistent metadata: data that persists after allocation (e.g. allocation size) 
- Epheremal metadata: data that only exists when in the free list (e.g. prev_chunk)
- chunk: The actual slice of memory given to the program

Not free lists utilizes all of the elements of the general chunk. Some free lists, like the ones this article is about, only store the next node in the epheremal data. Some of the allocator variants that we will examine don't have any persistent metadata 

All of the allocators that we will look at will be specialized for a single allocation size, which simplifies many of the implementation details. Mainly, it allows us to treat free lists as stacks, in which allocating only involves popping a chunk from the head of the stack and freeing pushes a chunk onto the stack.

The simplest type of free list, one which will serve as the basis for the fixed size allocator, is a 'free stack' operating in a single block of memory.

#Fixed Size Free Stack#

The simplest implementation of a free list, a free stack, allocates a single block of memory and allocates elements from the block. Allocating an element pops an chunk from the head of the free list and freeing an element pushes a chunk to the head of the list.

The implementation is very simple. The structs that we will use in this allocator are

```C
//!A chunk of memory in the list
union chunk {
    union chunk *next;
    char data[1]; //needs a size, char c[] not supported in unions
};

//!The struct holding all needed information for the free list
struct fixed_list {
    union chunk *head;
    void *data;
}
```

```fixed_list_malloc``` simply pops the head off the list

```C

void *fixed_list_malloc(struct fixed_list *from) {
    if(!from->head)
        return NULL;
    void *data = from->head->data;
    from->head = from->head->next;
    return data;
}
```
while ```fixed_list_free``` pushes the new head

```C
void fixed_list_free(struct fixed_list *from, void *freeptr) {
    if(freeptr) {
        ((chunk *)freeptr)->next = from->head;
        from->head = freeptr;
    }
}
```
All that's needed are some helpers

```C

#define ALIGN_TO sizeof(void *)
//Could be 16 bytes, or an allocator parameter

static size_t pad_size(size_t start_size) {
    //If your compiler supports variable length arrays, this could be
    //sizeof(union {void *stuff, char[start_size] data;});
    
    if(start_size < ALIGN_TO)
        return ALIGN_TO;
    else if (!(start_size % ALIGN_TO))
        return start_size;
    return start_size + (ALIGN_TO - (start_size % ALIGN_TO))
}

```
a creation function

```C
struct fixed_list create_fixed_list(size_t object_num, size_t object_size) {
    fixed_list retlist;

    //pad the data to ensure proper alignment on each allocation
    object_size = pad_size(object_size);

    retlist.data = malloc(object_num * object_size);
    retlist.head = retlist.data;

    if(retlist.data) {
        union chunk *cur_chunk = retlist.data;
        for(size_t i = 0; i < object_num - 1; i++) {
            cur_chunk->next = (char *)cur_chunk + object_size;
            cur_chunk = cur_chunk->next;
        }
        cur_chunk->next  = NULL;
    }
    return retlist;
}
```
and a deletion method

```C
struct fixed_list destroy_fixed_list(fixed_list *inlist) {
    free(retlist->data);
}

```
and we have a working (albeit rudimentary) allocator. 

blah blah here

#Free Stack#

The next step is to allow the allocator to allocate an arbitrary amount of memory. This can be done without a lot of extra work - we simply store a list of blocks while using the same malloc/free schema

This doesn't require many changes - we add the helper structure ```slab```

```C
struct slab {
    chunk *data;
    struct slab *next;
};
```
and modify the allocator structure to 

```C
struct list_alloc {
    slab *slabs;
    chunk *head;
    size_t object_size;
    size_t object_num;
};
```

The ```fixed_list_free``` function doesn't see any changes except for a name change to ```list_free```, and ```fixed_list_malloc``` only has a slight change to

```C
void *fixed_list_malloc(struct fixed_list *from) {
    if(!from->head)
        return add_slab(from);
    void *data = from->head->data;
    from->head = from->head->next;
    return data;
}
```

Instead of returning null when the allocator is out of memory, we add another slab and return a pointer from that.

```add_slab``` closely resembles the ```create_fixed_list``` function, since it serves a similar purpose

```C
static void *add_slab(struct list_alloc *from) {

    //pad the data to ensure proper alignment on each allocation
    size_t object_size = pad_size(from->object_size);

    void *data = malloc(object_num * object_size + sizeof(slab));

    if(data) {
        //skip first chunk since it is being returned
        union chunk *cur_chunk = (char *)data + object_size;
    
        for(size_t i = 0; i < from->object_num - 1; i++) {
            cur_chunk->next = (char *)cur_chunk + object_size;
            cur_chunk = cur_chunk->next;
        }
        cur_chunk->next = from->head;

        slab *newslab = (char *)cur_chunk + object_size;
        newslab->data = cur_chunk->next;
        newslab->next = from->slabs;

        from->slabs = newslab;
        from->head = cur_chunk;
    }
    return data;
}
```
