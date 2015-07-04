---
layout: post
title: Free lists and per-datastructure allocators
---

*Code and post in progress, [different version of code here](https://github.com/schets/fast_alloc)*

#Intro#

blah blah blah memory is important malloc is slow many datastructures are only accessed by one thread at a time cache locality write this later ...

...

...


#Fixed Size Free List#
[Free lists](https://en.wikipedia.org/wiki/Free_list) are one of the most convinient methods of creating an allocator, and are pretty easy to implement. They're used in some of the most common allocators used today, including [ptmalloc (glibc malloc)](http://code.woboq.org/userspace/glibc/malloc), [jemalloc](http://www.canonware.com/jemalloc/) and [tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html), as well as in many [specialized allocators](http://gameprogrammingpatterns.com/object-pool.html).

The fixed size free list only allocates memory chunks of a single size, allowing for a simple and fast free list implementation ([This is a good introduction free lists] (http://gameprogrammingpatterns.com/object-pool.html)). 

Our implementation will work similary, except it will store a list of memory blocks from which memory 'units' can be allocated so the size can change depending on allocation requirements.

The types that this allocator needs for storing/dispensing memory are

```C
//!Represents a single unit of memory to be dispensed by the allocator
union chunk {
    union chunk *next; //pointer to next chunk
    char data[1]; //placeholder for data value
};
 
//!Block of memory where chunks are held
struct slab {
    chunk *data; //pointer to beginning of data segment
    struct slab *next; //pointer to next slab in salb list
};
```

and the allocator will be defined as

```C
struct free_list {
    union chunk *head; //pointer to head of list, first open element
    struct slab *slabs; //pointer to list of memory blocks
    size_t object_size; //bytes that each object takes up (with padding)
    size_t object_num; //number of units each block should hold
};
```

Allocating and freeing with the list is very simple; most of the complexity lies in managing memory blocks. ```list_malloc``` simple pops an element from the head of the list and ```list_free``` pushes an element onto the top of the list

```C
void *list_malloc(struct free_list *from) {
    if (!from->head)
        return add_slab(from); //defined later
    void *data = from->head->data;
    from->head = from->head->next;
    return data;
}

void list_free(struct free_list *to, void *freeptr) {
    if (freeptr) {
        ((struct chunk*)freeptr)->next = from->head;
        from->head = freeptr;
    }
}
```

```add_slab``` attempts to allocate a new slab and construct an initial free list in it. It then returns a pointer to the first element of the list upon success or returns NULL otherwise

```C
static void init_slab(void *data, struct list_alloc *from) {
    //skip first chunk since it is being returned, set second chunk as new head
    union chunk *cur_chunk = (char *)data + from->object_size;
    from->head = cur_chunk

    //set each chunk to point to the following chunk in the list
    for(size_t i = 1; i < from->object_num - 1; i++) {
        cur_chunk->next = (char *)cur_chunk + from->object_size;
        cur_chunk = cur_chunk->next;
    }

    //set end of new list to point to the previous head (should generally be NULL)
    cur_chunk->next = from->head;

    //creat the slab and set it as the head of the slab list
    slab *newslab = (char *)cur_chunk + from->object_size;
    newslab->data = data;
    newslab->next = from->slabs;
    from->slabs = newslab;
}

static void *add_slab(struct list_alloc *from) {
    void *data = malloc(from->object_num * object_size + sizeof(slab));
    if(data) {
        init_slab(data, from);
    }
    return data;
}
```

The only other functions that need to be implemented are initialization and desctruction methods, which are quite simple

```C
//helper to ensure that the passed size will be aligned to a certain value
static size_t pad_size_to(size_t initial, size_t align) {
    if (initial <= align)
        return align;
    if ((initial % align) == 0)
        return initial;
    return initial + align - (initial % align);
}

void list_create(struct free_list *list, size_t object_size, size_t object_num) {
    if (list) {
        list->object_num = object_num;
        list->object_size = pad_size_to(object_size, sizeof(chunk));
        list->head = NULL;
        list->slabs = NULL;
    }
}

void list_destroy(struct free_list *ldel) {
    while(ldel->slabs) {
        struct slab *todel = ldel->slabs;
        ldel->slabs = ldel->slabs->next;
        free(todel->data);
    }
    ldel->slabs = NULL;
    ldel->head = NULL;
}
```

With not a lot of code, we've managed to make a fairly performant allocator - most allocations only pop an element form the list, and all frees simply push an element.
Furthermore, since elements are tightly packed in blocks, the returned data is much friendlier to the cache. The effect of this can be lessened as an instance accumulates more slabs and the chunks from each one are mixed in the list, but that doesn't always matter as much in practice - due to the way data is freed, allocations will usually return a recently accessed pointer anyways.
