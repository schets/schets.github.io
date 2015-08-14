---
layout: post
title: Fixed size free lists and per-datastructure allocators
---

I'm pushing this to master so that others can more easily review this. If anyone comes across this by chance there's no promise of quality in the post yet. But if I've said something completely wrong/stupid, feel free to let me know.

*Code and post in progress, [different version of code here](https://github.com/schets/fast_alloc)*.

Most general purpose allocators use single heap for allocating memory; some use threadlocal heaps.
While this works incredibly well for a general purpose allocator, it has some unfortunate downsides.
Despite the best efforts of modern allocators, allocated could come from any location in the heap.
In addition, there's often some extra memory attachd to each allocation for bookkeeping purposes

Locality of reference and predictable memory access, or lack thereof, play a
[huge role](http://igoro.com/archive/gallery-of-processor-cache-effects/) in the performance of a program.



#Fixed Size Free List#
[Free lists](https://en.wikipedia.org/wiki/Free_list) are one of (if not the) most effective tools for creating allocators. They're used in some of the most common allocators, including [ptmalloc (used in glibc)](http://code.woboq.org/userspace/glibc/malloc), [jemalloc](http://www.canonware.com/jemalloc/) and [tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html), as well as in many [specialized allocators](http://gameprogrammingpatterns.com/object-pool.html).

The fixed size free list only allocates memory chunks of a single size, allowing for a simple and fast free list implementation. [This is a good introduction] (http://gameprogrammingpatterns.com/object-pool.html) to the type of list that we'll be implementing. 

Our implementation will work similary, except it will store a list of memory blocks from which memory 'units' can be allocated so that the allocator can expand as needed.

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
    struct slab *next; //pointer to next slab in slab list
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

We've managed to make a fairly performant allocator -
on my laptop ```list_malloc```/```list_free``` pairs take ~2-5ns
while malloc/free pairs which take ~80 ns <a href="#fn2" id="ref2">[2]</a>.

Of course, this allocator only has a bit of the functionality of malloc - but that functionality isn't always needed.

Let's see how much this affects the creation and modification of a datastructure.

#Benchmarking a Binary Tree#

This benchmark is pretty simple - it will measure how fast it is to create and modify a binary tree of varying sizes uses each allocator.
Each binary tree will first be filled with (size/2) random numbers, and then for 

#Compacting Data#

Unlike malloc, the free list will allocate data from compact blocks of memory, and sequential allocations are initially contiguous (usually).
In theory, this should lead to better cache performance, since cache lines will be more likely
to contain relevant data, instead of junk or data not being used anytime soon.

Let's look at a benchmark to see how this can affect performance, both based on the binary tree.
This benchmark will generate many small trees of similar size using different allocation patterns.
The implementations that will be compared are:

1. Trees randomly built and then heavily pruned, allocated using malloc/free.
2. Trees copied from 1 allocated with malloc (as a control for 4 and 5)
3. Trees built in the same fashion as 1 but using a free list
4. Compact trees copied from 1 using a free list and allocations in an 'optimal' pattern
5. Compact trees copied from 1 using a free list with discontiguous allocations

The discontiguous allocations come from a free list that has been scrambled by allocating/deallocating elements in a random order

Here are the results:

<img src="{{site.baseurl}}/images/big-bench-copy-ns.png"/>

Awesome! Compacting the data has a serious effect on the access times!

Unfortunately, all isn't well in free list land. The trees built with free lists

#Downsides 

The main downside of this allocator is that it can't return memory to the system until destruction,
and as a result can't share memory with other allocators.
In the previous section, the performance discrepancy between the malloc and the free list
was because malloc can reuse the freed memory to build the other trees <a href="#fn3" id="ref3">[3]</a>.
The free lists, on the other hand, have a lot of memory sitting around doing sitting around taking up cache space.

<hr></hr>


<font size=2 id="fn1">1. The time to execute a single instruction is often less than one cycle - modern x64 processors utilize pipelines, out of order execution, and multiple execution units to increase the number of instructions executed per cycle, and decrease time to execute an instruction.<a href="#ref1" title="Jump back to footnote 1 in the text.">↩</a></font>

<font size=2 id="fn2">2. ```list_free``` and ```list_malloc``` each execute 6 instructions, have one *very* predictable branch, and rarely leave the L1 cache. The time spent in such a function will be heavily influenced by other factors, like the current state of the pipeline, OoO, etc. Etiher way, it probably won't take a significant amount of time (the benchmark loops/branchs took a much larger amount of time than then allocations).<a href="#ref2" title="Jump back to footnote 2 in the text.">↩</a></font>

<font size=2 id="fn3">3. The discrepancy is also a result of the implementation. Each tree of the sequence is built and pruned before the next, meaning that malloc can share the pruned memory. If each tree was built, and then each tree pruned, malloc performs similarly to the free list. On the other hand, if the plain free-list solution shares a list between all of the trees, it has notably better performance<a href="#ref3" title="Jump back to footnote 3 in the text.">↩</a></font>
