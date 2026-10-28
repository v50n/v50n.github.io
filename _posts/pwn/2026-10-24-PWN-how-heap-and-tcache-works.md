---
title: "Pwn - How heap and tcache works"
categories:
  - Pwn
tags:
  - Pwn
layout: post
---

Hi, 

Today I want to share what I've learned about heap and tcache. Some years ago, when I learned about programming in C, I though heap is easy to understand. Just some malloc, some free() and voilà. 

Maybe I underestimate the heap, maybe I just overvalue myself. 

So, in order to understand the heap, I create a very simple program. What that do ? 
malloc(0x20) 2 times, malloc(0x100) 1 time, then free() all of them, and malloc another time 0x20. My libc version is 2.43

If someone found an error in my blog or found a miss understand, fell free to contact me ! 

## The program

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    void *a = malloc(0x20);   // 1st malloc: heap gets initialized
    void *b = malloc(0x20);
    void *c = malloc(0x100);

    free(a);   // -> tcache (size 0x30) 
    free(b);
    free(c);   
    void *d = malloc(0x20);   // reuses b 
    return 0;
}
```


## First malloc


When we call a 1st malloc(), the heap auto created, and it size here is 0x21000. This size is not very important :) 
```diff
0x0000555555559000 0x000055555557a000 0x0000000000000000 rw- [heap]
hex(0x000055555557a000-0x0000555555559000) = '0x21000'
```

Than look at gdb 

```diff
gef➤  x/60xg 0x0000555555559000
+0x555555559000:	0x0000000000000000	0x0000000000000031
+0x555555559010:	0x0000000000000000	0x0000000000000000
+0x555555559020:	0x0000000000000000	0x0000000000000000
+0x555555559030:	0x0000000000000000	0x0000000000020fd1
+0x555555559040:	0x0000000000000000	0x0000000000000000
+0x555555559050:	0x0000000000000000	0x0000000000000000
+0x555555559060:	0x0000000000000000	0x0000000000000000
```

### head of the heap
In this example, the related information is stored from the adresse 0x555555559030. The value here is 

```diff
+0x555555559030   0x0000000000000000	0x0000000000020fd1
```

It juste calcluate the remaining size of the heap : 0x21000 - 0x30 = 0x20fd0 

### A chunk

#### Size of chunk
When we make malloc(0x20), the program auto allocate a chunk in heap. But the size is not 0x20 as expected. The size is 0x30. 
Because the size of a chunk = malloc_size + header_size
The header is always 0x10, so the chunk size is 0x20 + 0x10 = 0x30

But why we have the value 0x31 here ? In fact the lower bite :1 equivalent to PREV_INUSE. And this value is always egal to 1 when we start a heap (dont know why - this will be hardcoded in the code, But I'm not check this info) 

So with the header + 0x20 data, we got the chunk like this

```diff
+0x555555559000:	0x0000000000000000	0x0000000000000031
+0x555555559010:	0x0000000000000000	0x0000000000000000
+0x555555559020:	0x0000000000000000	0x0000000000000000
```

## Other malloc

Same way of calculate. nothing special here

```diff
gef➤  x/60xg 0x0000555555559000
0x555555559000:	0x0000000000000000	0x0000000000000031
0x555555559010:	0x0000000000000000	0x0000000000000000
0x555555559020:	0x0000000000000000	0x0000000000000000
+0x555555559030:	0x0000000000000000	0x0000000000000031 ==> chunk b
+0x555555559040:	0x0000000000000000	0x0000000000000000
+0x555555559050:	0x0000000000000000	0x0000000000000000
+0x555555559060:	0x0000000000000000	0x0000000000000111 ==> chunk c
+0x555555559070:	0x0000000000000000	0x0000000000000000
+0x555555559080:	0x0000000000000000	0x0000000000000000
+0x555555559070:	0x0000000000000000	0x0000000000000000
+0x555555559080:	0x0000000000000000	0x0000000000000000
+0x555555559090:	0x0000000000000000	0x0000000000000000
```

## First free

### About free()

When we call free(something)
- glibc generate a random key (tcache_key) (in my version, this is a random but generated in runtime)  
- if (e->key == tcache_key)--> double free detected. Stop. 
- if not, just free a chunk, put the next_addr and key to the data section of a chunk

### tcache_perthread_struct

For the 1st call of free, the program allocate for tcache_perthread_struct 
```c
#define TCACHE_MAX_BINS 64
...
...
typedef struct tcache_perthread_struct {
    char counts[TCACHE_MAX_BINS];       // or uint16_t counts[TCACHE_MAX_BINS]
    tcache_entry *entries[TCACHE_MAX_BINS];
} tcache_perthread_struct;
```
In sumary, this tcache_perthread_struct have an counts array, and the pointer entries, it point to the addresse of the chunk

Because in x86_64, a total tcache_perthread_struct = 0x280. Need to + 0x10 for header = 0x290
Because chunk sizes must be aligned to 0x20 bytes ---> the size = 0x300

the structure of tcache: 
- header : size : (0x10)
- counts array location: 0x5555555591b0 - 0x555555559240
- entries array location: 0x555555559250 
- First chunk pointer address when we free(a): 0x555555559010

```diff
gef➤  x/100xg 0x0000555555559000
0x555555559000:	0x0000000000000000	0x0000000000000031 ==> chunk a
+0x555555559010:	**0x0000000555555559**	**0xdfb52cfdeee2f594** ==> FWD PTR (0x0000000555555559) and KEY
0x555555559020:	0x0000000000000000	0x0000000000000000
0x555555559030:	0x0000000000000000	0x0000000000000031 ==> chunk b
0x555555559040:	0x0000000000000000	0x0000000000000000
0x555555559050:	0x0000000000000000	0x0000000000000000
...
...
...
+0x5555555591a0:	0x0000000000000000	0x0000000000000301 ==> tcache_perthread_struct chunk size & Lower bite (1) 
+0x5555555591b0:	0x00100010000f0010	0x0010001000100010
+0x5555555591c0:	0x0010001000100010	0x0010001000100010
+0x5555555591d0:	0x0010001000100010	0x0010001000100010
+0x5555555591e0:	0x0010001000100010	0x0010001000100010
+0x5555555591f0:	0x0010001000100010	0x0010001000100010
+0x555555559200:	0x0010001000100010	0x0010001000100010
+0x555555559210:	0x0010001000100010	0x0010001000100010
+0x555555559220:	0x0010001000100010	0x0010001000100010
+0x555555559230:	0x0010001000100010	0x0010001000100010
+0x555555559240:	0x0010001000100010	0x0000000000000000
+0x555555559250:	0x0000555555559010	0x0000000000000000 ==> chunk ptr addr . tcache[0]
...
...
...
0x5555555594a0:	0x0000000000000000	0x0000000000020b61 ==> head of heap. 
```


### Forward ptr (FD)

Here is a code in libC. Thanks GOD the comment is here ! 
```c
/* Safe-Linking:
   Use randomness from ASLR (mmap_base) to protect single-linked lists
   of TCache.  That is, mask the "next" pointers of the
   lists' chunks, and also perform allocation alignment checks on them.
   This mechanism reduces the risk of pointer hijacking, as was done with
   Safe-Unlinking in the double-linked lists of Small-Bins.
   It assumes a minimum page size of 4096 bytes (12 bits).  Systems with
   larger pages provide less entropy, although the pointer mangling
   still works.  */
#define PROTECT_PTR(pos, ptr) \
  ((__typeof (ptr)) ((((size_t) pos) >> 12) ^ ((size_t) ptr)))
#define REVEAL_PTR(ptr)  PROTECT_PTR (&ptr, ptr)
```

So the FD is calculate like this : 
```
FW PTR = (Addresse of pointer >> 12) xor NEXT_ADDR
```

In a chunk a:

FW PTR = (0x0000555555559010 >> 12) xor 0 # (NOT have next_addr yet) 
==> FW PTR = 0x0000555555559

### key
I dont know why I don't have the bk pointer (backward pointer). Instead of that, in the +0x08, I have got a random key generated. 
No problem :) 
It value is 0xdfb52cfdeee2f594 (a  very random) 

## 2th free (free b)

```diff
gef➤  x/150xg 0x0000555555559000
0x555555559000:	0x0000000000000000	0x0000000000000031 ==> chunk a
0x555555559010:	0x0000000555555559	0xdfb52cfdeee2f594
0x555555559020:	0x0000000000000000	0x0000000000000000
0x555555559030:	0x0000000000000000	0x0000000000000031 ==> chunk b
+0x555555559040:	0x000055500000c549	0xdfb52cfdeee2f594
0x555555559050:	0x0000000000000000	0x0000000000000000
...
...
...
0x5555555591a0:	0x0000000000000000	0x0000000000000301 
+0x5555555591b0:	0x00100010000e0010	0x0010001000100010 ==> Some weird calcul of array location.... Dont understand 
0x5555555591c0:	0x0010001000100010	0x0010001000100010
0x5555555591d0:	0x0010001000100010	0x0010001000100010
0x5555555591e0:	0x0010001000100010	0x0010001000100010
0x5555555591f0:	0x0010001000100010	0x0010001000100010
0x555555559200:	0x0010001000100010	0x0010001000100010
0x555555559210:	0x0010001000100010	0x0010001000100010
0x555555559220:	0x0010001000100010	0x0010001000100010
0x555555559230:	0x0010001000100010	0x0010001000100010
0x555555559240:	0x0010001000100010	0x0000000000000000
+0x555555559250:	0x0000555555559040	0x0000000000000000 ==> tcache[0] value change for 0x0000555555559040
...
...
...
0x5555555594a0:	0x0000000000000000	0x0000000000020b61 ==> head of heap. Nothing change
```

We can calculate manually to check the FW_PTR :) . And the key is always the same
```
FW_PTR = (Addresse of pointer >> 12) ^ NEXT_ADDR
FW_PTR = (0x0000555555559040 >> 12) ^ 0x555555559010 (next_addr of chunk a data) 
FW_PTR = 0x55500000c549 ! Voilà ! 
```

## 3th free (free c)

```diff
gef➤  x/150xg 0x0000555555559000
0x555555559000:	0x0000000000000000	0x0000000000000031
0x555555559010:	0x0000000555555559	0xdfb52cfdeee2f594
0x555555559020:	0x0000000000000000	0x0000000000000000
0x555555559030:	0x0000000000000000	0x0000000000000031
0x555555559040:	0x000055500000c549	0xdfb52cfdeee2f594
0x555555559050:	0x0000000000000000	0x0000000000000000
0x555555559060:	0x0000000000000000	0x0000000000000111 ==> chunk c
+0x555555559070:	0x0000000555555559	0xdfb52cfdeee2f594
0x555555559080:	0x0000000000000000	0x0000000000000000
0x555555559090:	0x0000000000000000	0x0000000000000000


0x5555555591a0:	0x0000000000000000	0x0000000000000301 ==> tcache
+0x5555555591b0:	0x00100010000e0010	0x0010001000100010
0x5555555591c0:	0x0010001000100010	0x000f001000100010
0x5555555591d0:	0x0010001000100010	0x0010001000100010
0x5555555591e0:	0x0010001000100010	0x0010001000100010
0x5555555591f0:	0x0010001000100010	0x0010001000100010
0x555555559200:	0x0010001000100010	0x0010001000100010
0x555555559210:	0x0010001000100010	0x0010001000100010
0x555555559220:	0x0010001000100010	0x0010001000100010
0x555555559230:	0x0010001000100010	0x0010001000100010
0x555555559240:	0x0010001000100010	0x0000000000000000
0x555555559250:	0x0000555555559040	0x0000000000000000 ==> tcache[0]
...
+0x5555555592c0:	0x0000555555559070	0x0000000000000000 ==> NEW ! tcache[15]
...
...
...
0x5555555594a0:	0x0000000000000000	0x0000000000020b61 ==> head of heap. Nothing change
```

When you free() a chunk of size 0x110 (from malloc(0x100)), libc computes its bin index as (0x110 - 0x20) / 0x10 = 15 and pushes the chunk onto tcache[15], which is why a new value appears there. Each size class has its own bin, so only frees of that exact size touch tcache[15]


## malloc(0x20)
```diff
gef➤  x/150xg 0x0000555555559000
0x555555559000:	0x0000000000000000	0x0000000000000031 ==> chunk a
0x555555559010:	0x0000000555555559	0xd6b2dd7f837cd77c
0x555555559020:	0x0000000000000000	0x0000000000000000
0x555555559030:	0x0000000000000000	0x0000000000000031 ==> chunk b
+0x555555559040:	0x000055500000c549	0x0000000000000000 ==> Key is set to 0
0x555555559050:	0x0000000000000000	0x0000000000000000
...
...
...
0x555555559170:	0x0000000000000000	0x0000000000000301 ==> tcache
0x555555559180:	0x00100010000f0010	0x0010001000100010
0x555555559190:	0x0010001000100010	0x000f001000100010
0x5555555591a0:	0x0010001000100010	0x0010001000100010
0x5555555591b0:	0x0010001000100010	0x0010001000100010
0x5555555591c0:	0x0010001000100010	0x0010001000100010
0x5555555591d0:	0x0010001000100010	0x0010001000100010
0x5555555591e0:	0x0010001000100010	0x0010001000100010
0x5555555591f0:	0x0010001000100010	0x0010001000100010
0x555555559200:	0x0010001000100010	0x0010001000100010
0x555555559210:	0x0010001000100010	0x0000000000000000
+0x555555559220:	0x0000555555559010	0x0000000000000000 ==> tcache[0] .addr of chunk malloc(0x20) now is  0x0000555555559010
...
0x555555559290:	0x0000555555559070	0x0000000000000000
```


Because when we malloc(0x20), so the program look for tcache[0], get the addr 0x0000555555559040. It will go to the addr 0x0000555555559040 and set the key to 0x0000000000000000. Then recalculate the value of tcache[0]. 
```
Because : FW_PTR = (ADDR >> 12) ^ Next_addr
So: 
Next_addr = FW_PRT ^ (ADDR >> 12)
Next_addr = 0x000055500000c549 ^ (0x0000555555559040 >> 12)
>>> hex(0x000055500000c549 ^ (0x0000555555559040 >> 12))
'0x555555559010'
```
EZ right ? 

## Conclusion
I learn about how heap and tcache works. What is chunk, FD, key. How the FD is calculate. What is a structure of tcache, what is a structure of a heap. 
I'm very satisfy.  