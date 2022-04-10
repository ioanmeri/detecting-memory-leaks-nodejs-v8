# V8 Garbage collector

## Garbage Collector

It's all about memory

- What is a garbage collector

- What for is responsible

- What are the essential things of GC

### What is Garbage Collector (GC)

In Computer Science, Garbage Collection (GC) is a form of automatic memory management. The garbage collector **attempts to reclaim memory which was allocated by the program, but is no longer referenced** - also called garbage.

Garbage collection was invented by the american computer scientist John McCarthy around 1959 to simplify manual memory management in LISP

### What for GC is responsible

GC concentrated itself around the heap (dynamic allocated memory)

GC handle

- objects
- array
- primitives
- closures

GC does not handle:

- Resources other than memory
- network sockets
- database handles
- file and device descriptors

### Basic Principles

The basic principles of garbage collection are to find data objects in a program that cannot be accessed in the future, and to reclaim the resources used by those objects.

### Any Garbage Collector has a few essential tasks that it has to do periodically

- Find live / dead objects
- Recycle / reuse the memory occupied by dead objects
- Compact / defragment memory (optional)

### Advantages & Disadvantages

Pros

- no need to think when to free memory
- reduces number of errors we can make handling the memory

Cons

- GC need to do a full scavenge to know what object is not needed,
  event the programer may know it already
- It takes time to the main thread
- You don't know when GC will run

## V8 memory structure

V8 creates one Resident Set per process, which is divided into two main parts:

- Stack
- Heap

### Stack

Is the place where the static allocations have place, included your function name, function frames, primitive values and pointers to object. Basically all the data needed to run your application. Not managed by garbage collector, managed by the OS.

### Heap

Dynamically allocated part of the memory and it divides into seven different spaces:

- Cell space
- Map space
- Property space
- Large objects
- Code

are spaces used by the V8 engine, GC does not handle them

Also:

- Young generation
  - from space
  - to space
- Old generation

#### Young generation

Most objects are allocated here. New-space is small and is designed to be garbage collected very quickly, independent of other spaces

#### Old generation

Contains most objects which may have pointers to other objects.
Most objects are moved here after surviving in new-space for a while

Later on will understand, what is the distance from the global objects, why objects are referenced and cannot be de-allocated, what prevents the GC from de-allocating and reclaiming back the memory

This is a set of finite space, it will run out eventually if there is some memory leak and GC cannot reclaim back the memory

## Minor GC - Scavenge

It is run on the Young generation of the heap, which is divided into two spaces, one always empty:

- from space
- to space

---

E.g. we have 5 objects 1,2,3,4,5 in from space and we want to allocate a 6th one, but there isn't enough space in the from space. This is what triggers the minor GC

- from space
  - 1,2,3,4,5
- to space

First minor GC checks which objects are still alive, meaning that they are reachable from the root object

E.g. objects **1,2,4 are alive** and are evacuated into to space and memory is compact, de-fragmented.

- from space
  - 3,5
- to space
  - **1,2,4**

Objects 3,5 are considered dead, we only search for live objects and the rest will be deleted

- from space
- to space
  - **1,2,4**,6

The objects that survived one cycle of garbage collector will be marked: 1,2,4

The we switch the name of the spaces:

- from space
  - **1,2,4**,6
- to space

---

We want to allocate a new object: 7

E.g. **now** objects **1,2,6 are reachable from the root** but because objects **1,2 are marked** and has survived first minor GC and now they will survived the second one, instead of being moved into the to space they will move to **Old generation** and 6 will be moved into the to space and marked as it has survived one cycle.

- to space
  - 6
- from space
  - 4
- Old generation
  - 1,2

Object 4 will be deleted, and from space will be empty, new object will be allocated and spaces will switch again:

- to space
  - 6,7
- from space
- Old generation
  - 1,2

---

After each evacuation, the pointers of the stack are also updated to point in the new space and the objects are evacuated just by copying them. So all the bytes from the from space is copied into the to space and V8 engine uses the generational heap layout (most of the objects will be short living)

### Minor GC definition

A generational garbage collection strategy is well suited to an application that creates many short-lived objects, as is typical of many transactional applications.

### Orinoco

Most of these algorithms and optimizations are common in garbage collection literature and can be found in many garbage collected languages. But state-of-the-art garbage collection has come a long way. **One important metric for measuring the time spent in garbage collection is the amount of time that the main thread spends paused which GC is performed**. For traditional 'stop-the-world' garbage collectors, this time can really add up, and this time spent doing GC directly detracts from the user experience in the form of janky pages and poor rendering and latency.

## Major GC

Major GC in V8 starts with concurrent marking. As the heap approaches a dynamically computed limit, concurrent marking tasks are started. The helpers are each given a number of pointers to follow, and they mark each object they find as they follow all references from discovered objects. Concurrent marking happens entirely in the background while JavaScript is executing on the main thread. Write barriers are used to keep track of new references between objects that JavaScript creates while the helpers are marking concurrently.

Major GC is working on the **Old space**, trying to keep it clean and compact. Process is similar and is divided into three steps

1. Marking

The garbage collector identifies which objects are is use and which ones are not. The objects in use or reachable from known roots recursively are marked as alive. It's technically a depth-first-search of the heap which can be considered as a directed graph.

This is happening in a hubble thread and is not affecting the main thread so much.

2. Sweeping

The garbage collector traverses the heap and makes note of the memory address of any object that is not marked alive. This space is now marked as free in the free list and can be used to store other objects.

3. Compacting (optional)

After sweeping, if required, all the survived objects will be moved to be together. This will decrease fragmentation and increase the performance of allocation of memory to newer objects
