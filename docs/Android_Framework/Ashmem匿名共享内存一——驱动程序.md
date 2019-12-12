# Ashmem匿名共享内存一——驱动程序

### 概述

Ashmem，全称Anonymous Shared Memory。匿名共享内存。

Ashmem匿名共享内存是Android系统实现的一种内存共享机制，用于IPC。与传统的Linux内存共享一样，Ashmem也是基于tmpfs实现的，不过Ashmem做了更为细致的内存管理，将整块共享内存切割，可动态回收，动态分配。

### 数据结构

#### ashmem_area

用来描述一块共享内存。

* 
  name：共享内存的名称。每一个块共享内存的名称都以dev/ashmem开头，如果没有指定名称，则名字就是dev/ashmem。
* 
  unpinned_list：描述一个解锁内存块列表。ashmem将整个共享内存分为多个小内存块，小内存块有两种状态，一种是锁定，即正在使用，不可回收；一种是解锁，解锁状态下的小内存块可以被回收。这个列表下的地址块互不相交，且按地址值从大到小排列。
* 
  file：指向临时文件系统tmpfs中的一个文件。
* size：描述临时文件的大小。这个大小也就是匿名共享内存的大小。

  > tmpfs：Linux提供的一种临时文件系统，其中的文件存在于内存中，不在硬盘上，也会挂载在文件系统中，不过断电即消失。可以用df命令查看。
* 
  prot_mask：访问保护位。默认是exec|read|write。

#### ashmem_range

用来描述一小块解锁内存块。

* unpinned：用于链入宿主共享内存的unpinned列表中。
* lru：用于链入全局LRU链表ashmem_lru_list中，这个链表会在回收的时候用到。
* pgstart/pgend：描述这一小块内存的开始和结束地址，单位是页。
* purged：描述是否已经被回收。

#### ashmem_pin

作为IO控制命令ASHMEM_PIN和ASHMEM_UNPIN的参数，也即解锁和锁定时的参数。成员变量描述了起始地址和长度。

### 初始化

驱动程序以misc设备文件的形式存在文件系统中，而每块共享内存则以tmpfs的形式存在文件系统中。

* 创建两个slab缓冲区分配器，分别用来分配ashmem_area和ashmem_range。

  > slab：缓冲区分配器。虚拟内存的内存分配是以页为单位，但有些小对象所需的内存空间比页大小要小得多，为其分配一整个页是效率低下的。slab分配器持有以页为单位的内存空间，而通过它可以将页再次分割，分配给小对象。
* 调用misc_register注册一个匿名共享内存设备。ashmem被当做一个misc设备，以设备文件/dev/ashmem的形式放在文件系统中。应用程序通过它来访问Ashmem驱动程序。

  > misc：意为杂乱的设备。Linux中专门用来管理特殊设备或者不知如何分类的设备，这些设备公用一个主设备号，以次设备号区分。

  ashmem_misc结构体正是ashmem设备文件在内存中的表现形式，其指定了设备文件名字、设备文件操作表（包含打开、关闭、映射、IO控制）。

* 
向操作系统内存管理机制注册一个内存回收函数ashmem_shrinker。当系统内存不足时会调用这个函数，这会回收处于解锁状态的小内存块。

### 设备文件打开

根据ashmem_misc中的操作表的指定，对设备文件的操作最终都会映射到ashmem.c中实现的各个函数。打开对应了ashmem_open。用户进程调用的是open，返回一个file，而ashmem_open中的参数file就是这个返回值。

* 从slab缓冲区中分配一个ashmem_area结构体。做unpinned_list、名称（/dev/ashmem/  ）、访问保护位的初始化。然后将其赋给file的private_data。

### 内存映射

当应用程序调用mmap将**设备文件/dev/ashmem**映射到地址空间时，Ashmem会创建一个临时文件。

* 根据文件操作表的映射关系，mmap会被映射到ashmem_mmap，传进来的参数是打开时返回的file，和一个描述内存的结构体vma。
* 首先从file中获取ashmem_area。并做大小、访问标记位的判断。
* 创建临时文件。调用shmem_file_setup在临时文件tmpfs中创建一个临时文件，打开的文件保存在ashmem_area的成员变量file中。
* 将vma的映射文件设置为刚刚创建出来的临时文件，设置其内存操作表，主要改变其缺页方法fault为shmem_fault。其逻辑为：缺页时，先查页面缓冲区，如果有，则映射到缺页的虚拟地址；如果没有到换出设备中查，查到了添入缓冲区；如果再没有，分配新的物理页面，添入缓冲区。通过这种方式，则使用同一个file结构体，就是同一个页面缓冲区，可以映射到不同的vma中，实现了共享内存。

### 锁定和解锁

锁定和解锁主要是通过IO控制命令ASHMEM_PIN和ASHMEM_UNPIN来实现解锁。最终映射到ashmem_pin和ashmem_unpin。

* 解锁：宿主共享内存的unpinned_list，按地址从大到小排列，而且互不相交，所以在解锁的时候，要查询顺序，判断是否相交，如果相交要合并；以此得出pgstart和pgend，然后通过slab分配一个ashmem_range结构体。再添入宿主列表unpinned_list和全局列表ashmem_lru_list中。
* 锁定：从unpinned_list和ashmem_lru_list中删除。

### 回收

回收函数ashmem_shrink在驱动程序初始化时就注册好了，当操作系统内存不足时，会调用这个函数，回收。回收的对象就是全局列表ashmem_lru_list中的解锁内存块。

## 总结

* Android系统提供Ashmem匿名共享内存机制，提供驱动程序，驱动程序以misc设备文件的形式存在系统中。命名为/dev/ashmem。
* 用户进程通过文件操作/dev/ashmem设备文件，访问Ashmem驱动程序，如打开、关闭、映射等等。
* 每块匿名共享内存对应一个临时文件，临时文件基于tmpfs实现，文件则相当于内存空间，其具有页面缓冲区等等内存结构。多个应用进程访问时，使用的是同一个页面缓冲区，以此实现了内存共享。
* 匿名共享内存对自己做了切割，将空闲的内存区域描述出来，注册给操作系统，当操作系统内存不足时，会回收这部分内存。
