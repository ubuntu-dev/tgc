Tiny Garbage Collector
======================

About
-----

`tgc` is a tiny garbage collector written in ~500 lines of C and based on the 
[Cello Garbage Collector](http://libcello.org/learn/garbage-collection).

```c
#include "tgc.h"

static tgc_t gc;

static void example_function() {
  char *message = tgc_alloc(&gc, 64);
  strcpy(message, "No More Memory Leaks!");
}

int main(int argc, char **argv) {
  tgc_start(&gc, &argc);
  
  example_function();

  tgc_stop(&gc);
}
```

Usage
-----

`tgc` is a conservative, thread local, mark and sweep garbage collector,
which supports destructors, and automatically frees memory allocated by 
`tgc_alloc` and friends after it becomes _unreachable_.

A memory allocation is considered _reachable_ by `tgc` if...

* a pointer points to it, located on the stack at least one function call 
deeper than the call to `tgc_start`, or,
* a pointer points to it, inside memory allocated by `tgc_alloc` 
and friends.

Otherwise a memory allocation is considered _unreachable_.

Therefore some things that _don't_ qualify an allocation as _reachable_ are, 
if...

* a pointer points to an address inside of it, but not at the start of it, or,
* a pointer points to it from inside the `static` data segment, or, 
* a pointer points to it from memory allocated by `malloc`, 
`calloc`, `realloc` or any other non-`tgc` allocation methods, or, 
* a pointer points to it from a different thread, or, 
* a pointer points to it from any other unreachable location.

Given these conditions, `tgc` will free memory allocations some time after 
they become _unreachable_. To do this it performs an iteration of _mark and 
sweep_ when `tgc_alloc` is called and the number of memory allocations exceeds 
some threshold. It can also be run manually with `tgc_run`.

Memory allocated by `tgc_alloc` can be manually freed with `tgc_free`, and 
destructors (functions to be run just before memory is freed), can be 
registered with `tgc_set_dtor`.


Reference
---------

```c
void tgc_start(tgc_t *gc, void *stk);
```

Start the garbage collector on the current thread, beginning at the stack 
location given by the `stk` variable. Usually this can be found using the 
address of any local variable, and then the garbage collector will cover all 
memory at least one function call deeper.

* * *

```c
void tgc_stop(tgc_t *gc);
```

Stop the garbage collector and free its internal memory.

* * *

```c
void tgc_run(tgc_t *gc);
```

Run an iteration of the garbage collector, freeing any unreachable memory.

* * * 

```c
void *tgc_alloc(gc_t *gc, size_t size);
```

Allocate memory via the garbage collector to be automatically freed once it
becomes unreachable.

* * *

```c
void *tgc_calloc(gc_t *gc, size_t num, size_t size);
```

Allocate memory via the garbage collector and initalise it to zero.

* * *

```c
void *tgc_realloc(gc_t *gc, void *ptr, size_t size);
```

Reallocate memory allocated by the garbage collector.

* * *

```c
void tgc_free(gc_t *gc, void *ptr);
```

Manually free an allocation made by the garbage collector. Runs any destructor
if registered.

* * *

```c
void *tgc_alloc_opt(tgc_t *gc, size_t size, int flags, void(*dtor)(void*));
```

Allocate memory via the garbage collector with the given flags and destructor.

For the `flags` argument, the flag `TGC_ROOT` may be specified to indicate that 
the allocation is a garbage collection _root_ and so should not be 
automatically freed and instead will be manually freed by the user with 
`tgc_free`. Because roots are not automatically freed, they can exist in 
normally unreachable locations such as in the `static` data segment or in 
memory allocated by `malloc`. Otherwise the `flags` argument can be set to 
zero.

The `dtor` argument lets the user specify a _destructor_ function to be run 
just before the memory is freed. Destructors have many uses, for example they 
are often used to automatically release system resources (such as file handles)
when a data structure is finished with them. For no destructor the value `NULL` 
can be used.

* * *

```c
void *tgc_calloc_opt(tgc_t *gc, size_t num, size_t size, int flags, void(*dtor)(void*));
```

Allocate memory via the garbage collector with the given flags and destructor 
and initalise to zero.

* * *

```c
void tgc_set_dtor(tgc_t *gc, void *ptr, void(*dtor)(void*));
```

Register a destructor function to be called after the memory allocation `ptr`
becomes unreachable, and just before it is freed by the garbage collector.

* * *

```c
void tgc_set_flags(tgc_t *gc, void *ptr, int flags);
```

Set the flags associated with a memory allocation, for example the value 
`TGC_ROOT` can be used to specify that an allocation is a garbage collection 
root.

* * *

```c
int tgc_get_flags(tgc_t *gc, void *ptr);
```

Get the flags associated with a memory allocation.

* * *

```c
void(*tgc_get_dtor(tgc_t *gc, void *ptr))(void*);
```

Get the destructor associated with a memory allocation.

* * *

F.A.Q
-----

### `tgc` isn't working when I increment pointers!

Due to the way `tgc` works, it always needs a pointer to the start of each 
memory allocation to be reachable. This can break algorithms such as the 
following, which work by incrementing a pointer.

```c
void bad_function(char *y) {
  char *x = tgc_alloc(&gc, strlen(y) + 1);
  strcpy(x, y);
  while (*x) {
    do_some_processsing(x);
    x++;
  }
}
```

Here, when `x` is incremented, it no longer points to the start of the memory 
allocation made by `tgc_alloc`. Then during `do_some_processing`, if a sweep 
is performed, `x` will be declared as unreachable and the memory freed.

If the pointer `x` is also stored elsewhere such as inside a heap structure 
there is no issue with incrementing a copy of it - so most of the time you 
don't need to worry, but occasionally you may need to adjust algorithms which
do significant pointer arithmetic. For example, in this case the pointer can be 
left as-is and an integer used to index it instead:

```c
void good_function(char *y) {
  int i;
  char *x = tgc_alloc(&gc, strlen(y) + 1);
  strcpy(x, y);
  for (i = 0; i < strlen(x); i++) {
    do_some_processsing(&x[i]);
  }
}
```

For now this is the behaviour of `tgc` until I think of a way to 
deal with offset pointers nicely.


### `tgc` isn't working when optimisations are enabled!

Variables are only considered reachable if they are one function call shallower 
than the call to `tgc_start`. If optimisations are enabled somtimes the 
compiler will inline functions which removes this one level of indirection.

The most portable way to get compilers not to inline functions is to call them 
through `volatile` function pointers.

```c
static tgc_t gc;

void please_dont_inline(void) {
  ...
}

int main(int argc, char **argv) {
  
  tgc_start(&gc, &argc);

  void (*volatile func)(void) = please_dont_inline;
  func();
  
  tgc_stop(&gc);

  return 1;
}
```

### Why do I get _uninitialised values_ warnings with Valgrind?

The garbage collector scans the stack memory and this naturally contains 
uninitialised values. It scans memory safely, but if you are running through 
Valgrind these accesses will be reported as warnings/errors. Other than this 
`tgc` shouldn't have any memory errors in Valgrind, so the easiest way to 
disable these to examine any real problems is to run Valgrind with the option 
`--undef-value-errors=no`.

### Is this real/safe/portable?

Definitely! While there is no way to create a _completely_ safe/portable 
garbage collector in C this collector doesn't use any platform specific tricks 
and only makes the most basic assumptions about the platform, such as that the 
architecture using a continuous call stack to implement function frames.

It _should_ be safe to use for more or less all reasonable architectures found 
in the wild and has been tested on Linux, Windows, and OSX, where it was easily 
integrated into several large real world programs (see `examples`) such as 
`bzip2` and `oggenc` without issue.

Saying all of that, there are the normal warnings - this library performs 
_undefined behaviour_ as specified by the C standard and so you use it at your 
own risk - there is no guarantee that something like a compiler or OS update 
wont mysteriously break it.

