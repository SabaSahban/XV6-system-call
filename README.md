# Adding kfreemem() system call:

Our goal is to implement a system call for returning free memory space. (number of free pages available in the kernel).

## Implementation


The xv6 kernel keeps track of free pages in a linked list.
The kernel uses the struct run to track a free page. This structure stores a pointer to the next free page and is stored within the page itself.

```c
struct run {
  struct run *next;
};
```
The kernel keeps a pointer to the first page of this free list in the structure struct kmem. Pages are added to this list upon initialization, or on freeing them up.

```c
struct {
  struct spinlock lock; 
  struct run *freelist;
} kmem;
```

Functions related to page allocation can be found in kalloc.c file.
First, we will write a function to determine the number of free pages.
freepages() iterates through freelist linked list in order to find the number of free pages. (implemented in kalloc.c)
We should acquire the kmem.lock while counting to avoid other processes from modifying the freelist.

```c
int
freepages() {
  int pages = 0;
  struct run *r;
  acquire(&kmem.lock);
  r = kmem.freelist;
  
  //iterate over linked list
  while (r != 0) {
   pages += 1;
   r = r->next; 
  }
  
  release(&kmem.lock); return pages;
}
```
After adding this function to kalloc.c we also need to add the freepages() prototype to def.s.h:
```c
// kalloc.c
void* kalloc(void);
void kfree(void *);
void kinit(void);
int freepages(void);
```
After adding this function to kalloc.c we should add the freepages() function to kernel/main.c to print the number of free pages on boot, so we can make sure our function working.
```c
void
main() {
  if(cpuid() == 0){
  consoleinit();
  printfinit();
  printf("\n");
  printf("xv6 kernel is booting\n"); 
  printf("\n");
  kinit(); // physical page allocator 
  printf(“free memory pages: %d\n", freepages());
```
Result after running:

    $ make qemu 

xv6 kernel is booting
free pages: 32734

# Adding the kfreemem() system call

In kernel directory, we should add sys_kfreemem() to sysproc.c:

```c
uint64
sys_kfreemem(void) {
  return freepages();
}
```

Then we need to add the sys_kfreemem syscall to syscall.h where a number is assigned to every system call.

```c
#define SYS_kfreemem 22
```

Then, modify syscall.c to add sys_pages to the syscall table:
The function prototype which needs to be added to syscall.c file is as follows
```c
extern uint64 sys_kfreemem(void);
```
we need to add pointer to system call in syscall.c file
```c
[SYS_pages] sys_kfreemem,
```

# Adding the kfreemem() system call to the user directory
We should add entry kfreemem() at the end of usys.pl for the assembly functions that make the system calls to generate.
```c
entry("kfreemem");
```
Then we should add the prototype for kfreemem() to user.h
```c
int kfreemem(void);
```
# Adding a pages user-level program
We should add the user program to call kfreemem() and print the result:
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

Int main(int argc, char *argv[]) {
  printf("'running user program' free pages: %d\n", kfreemem());
  exit(0); 
}
```
For the final step, we should add our user program to Makefile:

```Makefile
UPROGS=\
 $U/_kfreemem\
```

Final result after running “make qemu”:
xv6 kernel is booting

free pages: 32734
$ kfreemem 
'running user program' free pages: 32764

# Output Analysis
Qemu allocates 128Mb of memory. And every memory page is 4096 bytes. So there’s 4096 * 32734 = 134078464 bytes of memory left.
It’s close to 128 Mb and close to our expectation because there are no processes running.
The user program output shows fewer memory pages because the init process has been created, and memory page has been allocated.
