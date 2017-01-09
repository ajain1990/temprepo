# Zero Copy Support

## Introduction
"Zero-copy" describes computer operations in which the CPU does not perform the
task of copying data from one memory area to another. This is used to save CPU
cycles and memory bandwidth.

There are different techniques for implementing this concept[1]. One way is to
use mmap for mapping the kernel buffer directly in user space virtual memory
area instead of copying it into the user buffer. This way, the user application
not only saves memory, it avoids data copy to and from user mode.
Establishment of this kind of mapping is also a costly VM operation that
requires page table modifications and TLB flushes in order to maintain memory
coherence, however since mapping is usually done for a relatively large area
(many kilobytes), these costs would be easily outweighted by CPU copy over
the same length.

### How memory management is performed in linux
Linux processes are implemented in the kernel as instances of task_struct
(process descriptor). The mm field in task_struct points to the memory
descriptor mm_struct, which is an executive summary of a program’s memory.
It stores the start and end of memory segments, the number of physical
memory pages used by the process, the amount of virtual address space used
etc. Within the memory descriptor we also find the two work horses for managing
program memory: the set of virtual memory areas and the page tables.

Each virtual memory area (VMA) is a contiguous range of virtual addresses,
these areas never overlap. An instance of vm_area_struct fully describes a
memory area, including its start and end addresses, flags to determine access
rights, behaviors and vm_file field to specify which file is being mapped by
the area, if any. A VMA that does not map a file is anonymous. Each memory
segment of a process’s memory e.g.:- heap, stack corresponds to a single VMA,
When you read file /proc/pid_of_process/maps, the kernel is simply going through
the linked list of VMAs for the process and printing each one.

## Support of zero copy in BUSE
Currently whenever an IO request comes, from the file system or any other IO
tool, on starling device, it is queued inside the respective device’s request
queue. Our user land daemon hsctld continuously probe buse for any incoming IO
request. Once it finds any it allocates a buffer to accommodate request data.
If it is a write request then pages of different segments of request are first
pinned in kernel space and then copied to allocated user land buffer with the
help of copy_to_user(). After this buffer will be sent to ceph by hctld for
actual write on the disk. On the other hand in case of read, buffer is handed
over to ceph which will issue actual read query to disk and requested data will
be filled in the buffer. Once control comes back to hsctld it issues IOCTL
in BUSE for completing request. Function corresponding  to this IOCTL firsts
pins pages of request’s segments in kernel space and the passed buffer data
would be copied in them with help of copy_from_user().  So overall in any of the
read/write request, we are making an extra copy of data.

Here's a diagram of what happens for write request:

The zero-copy idea is that whenever a hsctld pulls an IO request from buse a new
virtual memory area(VMA), having size equal to requested data length, is created
in process address space. Once the VMA is ready we would map all the requested
pages in it with the help of vm_insert_pfn(), and then the  starting address of
VMA would be sent to hsctld. Now ceph will access these pages directly for
read/write request. This would cut out an extra copy of request data buffer
which we are making currently and it would help in improving overall
IO performance.

## Descriptions of used APIs
### vm_mmap()
This function is used by the kernel to create a new linear address interval.
If the created address interval is adjacent to an existing address interval,
and if they share the same permissions, the two intervals are merged into one.
If this is not possible, a new VMA is created. It is declared in <linux/mm.h>:

```
unsigned long vm_mmap(struct file *file, unsigned long addr, unsigned long len, unsigned long prot, unsigned long flag, unsigned long offset)
```

It maps the file specified by file at given offset and length. The file parameter
can be NULL and offset can be zero, in which case the mapping will not be backed
by a file and called an anonymous mapping. If a file and offset are provided,
the mapping is called a file-backed mapping. The addr optionally specifies the
initial address from which to start the search for a free interval.
The prot parameter specifies the access permissions for pages in the memory area.

If any of the parameters are invalid, vm_mmap() returns a negative value.
Otherwise, a suitable interval in virtual memory is located. If possible,
the interval is merged with an adjacent memory area. Otherwise, a new
vm_area_struct structure is allocated and added to the address space’s
linked list and red-black tree of memory areas.

- mmap() Flags


--------------------------------------------------------------
| Value         | Discription                                 |
|---------------|---------------------------------------------|
| MAP_ANONYMOUS | Create an anonymous mapping                 |
| MAP_PRIVATE   | Modifications to mapped data are private    |
| MAP_FIXED     | Interpret addr argument exactly             |
| MAP_SHARED    | Modifications to mapped data are visible to |
|               | other processes mapping the same region.    |

- Memory protection values

-----------------------------------------------------------
| Value      | Discription                                 |
|------------|---------------------------------------------|
| PROT_READ  | The contents of the region can be read      |
| PROT_WRITE | The contents of the region can be modified  |
| PROT_EXEC  | The contents of the region can be executed  |

### vm_munmap()
The do_munmap() function removes an address interval from a specified process
address space. The function is declared in <linux/mm.h>:

```
int vm_munmap(unsigned long start, size_t len)
```
The first parameter specifies the address space from which the interval starting
at address start of length len bytes is removed. On success, zero is returned.
Otherwise, a negative error code is returned.

### find_vma()
The kernel provides a function, find_vma(), for searching the VMA in which a
given memory address resides. It is defined in mm/mmap.c:

```
struct vm_area_struct * find_vma(struct mm_struct *mm, unsigned long addr);
```

This function searches the given address space for the first memory area whose
vm_end field is greater than addr. It finds the first memory area that contains
addr or begins at an address greater than addr. If no such memory area exists,
the function returns NULL. Otherwise, a pointer to the vm_area_struct structure
is returned.

### vm_insert_pfn()
This function inserts single page frame number into user vma. Memory address,
virtual/physical, is divisible into a page number and an offset within the page.
If you discard the offset and shift the rest of an offset to the right, the
result is called a page frame number (PFN). As this function is called only
for pages that do not currently exist, we do not need to flush old virtual
caches or the TLB.

```
int vm_insert_pfn(struct vm_area_struct * vma, unsigned long addr, unsigned long pfn)
```

- vm_insert_pfn() arguments

-------------------------------------------------
| Argument  | Discription                       |
|-----------|-----------------------------------|
| vma       | User vma to map to                |
| addr      | Target user address of this page  |
| pfn       | Physical address of kernel memory |

## Future Work
Currently we are creating a new VMA for every IO request and deleting it once it
completes. Creation and deletion of VMA for every request is still hampering IO
performance. We can overcome this issue with the following mention approaches:
- We can use adaptive approach while serving IO requests in buse i.e. small
requests can be served by traditional way where we copy data in user land
allocated buffer.
- We can devise a way by which instead of creating a new VMA each time,
we can reuse existing VMAs if request size is less than or equal to VMA length.

## References
[1] ZeroCopy: Techniques, Benefits and Pitfalls [pdf]
(https://pdfs.semanticscholar.org/4931/bbb1b96d10f89b7406bb38757dcc2a10a435.pdf)

[2] Zero Copy I: User-Mode Perspective [link]
(http://www.linuxjournal.com/article/6345)

[3] The Linux Programming Interface Book by Michael Kerrisk
