# Ashmem匿名共享内存二——Runtime、Native、Framework

## Runtime中的cutils

提供了五个C接口来访问驱动程序。分别是`ashmem_create_region`创建、`ashmem_pin_region`锁定、`ashmem_unpin_region`解锁、`ashmem_set_prot_region`设置保护位、`ashmem_get_size_region`设置大小。

>  region：区域

### create

* 调用文件访问系统调用open，打开设备文件/dev/ashmem。会调用ashmem_open，创建一个ashmem_area结构体，返回一个文件描述符fd。
* 通过IO控制命令，为fd对应的匿名共享内存设置名字。
* 通过IO控制命令，给fd对应的匿名共享内存设置大小。
* 返回fd。

### pin/unpin

通过IO控制命令，锁定或解锁一小块内存空间，传递的参数是fd、ashmem_pin。

### set prot

通过IO控制命令，设置访问保护位。

###　get size

通过IO控制命令，获取匿名共享内存的大小。

## Native层C++接口

主要提供两个类，MemoryHeapBase和MemoryBase。

MemoryHeapBase是整块内存的共享，MemoryBase是共享部分内存。

这两个类都代表的是Service组件，其实就是一套写好的Binder。

### MemoryHeapBase

#### IMemoryHeap

IMemoryHeap类定义了接口，主要有四个函数：`getHeapID`获取文件描述符、`getBase`

获取映射地址、`getSize`获取大小、`getFlags`获取保护位。

在Binder通讯中，Client端和Server端用的是同一个接口。

##### Server端

通过Binder进行通讯，也就是Binder本地对象。

###### BnMemoryHeap

只有onTransact，做中转。转到MemroyHeapBase。

###### MemoryHeapBase

实现了IMemoryHeap接口。本身成员变量有：mFD文件描述符、mSize大小、mBase映射地址、mFlags访问保护位。

+ 构造函数
  * 调用ashmem_create_region，创建匿名共享内存，返回文件描述符，也就是mFD。这里会传入名字、大小、访问保护位。
  + mapfd，将内存块映射到本进程的地址空间。这里会调用到ashmem_mmap，会创建一个临时文件，即代表匿名共享内存。返回的值是映射的地址mBase。

+ 接口函数

  就是返回成员变量

##### Client端

###### BpMemroyHeap

实现了IMemoryHeap。当Client进程第一次访问这个代理对象时，才会请求Server端返回匿名共享内存的信息。这个类在构造函数中只是赋初值，请求都是在接口实现中做的。

+ 请求
  * 获取代理对象。Binder库为Client端维护了一个缓存，如果多个代理应用同个服务，返回的是同一个代理对象。如果代理对象已经存在，则返回，如果不存在，则创建返回。这个代理对象只记录了信息，并没有具体的实现。
  + 请求。以代理对象的InterfaceDescriptor，通过Binder驱动，发送给服务端的对象，返回整个共享内存的信息，保存在本地。
+ 映射
  * 请求得到的共享内存的信息，调用mmap映射到本进程的空间。
+ 访问
  * 请求得到的共享内存的信息中有mBase，即一个映射地址，通过该地址则可以访问。

### MemoryBase

就是基于MemeoryHeapBase，加了偏移量和长度。其内部持有一个MemoryHeapBase实例，而其本身描述的就是这片内存的一小部分，以偏移量和长度表示。

#### Server端

##### IMemory

定义了MemoryBase的服务接口，有四个接口函数：`getMemory`获取内部的MemoryHeapBase、`pointer`获取MemoryBase描述的一小段内存、`size`大小、

`offset`偏移量。

* pointer：获取内部的mHeap，然后加上偏移量返回，则可以得到那部分内存。

##### MemoryBase

* getMemory：设置好offset和size，返回heap。

#### Client端

##### BpMemroy

通过Binder和Server端通讯，可以获取mHeap和偏移量。则这个地址可以访问。

## Framework

Android对Ashmem共享内存提供的Java接口主要有一个类，即MemoryFile。

### MemoryFile

封装了文件描述符、地址、保护位等等。

#### 构造方法

有两种，一个是传入名称和长度，创建匿名共享内存，做映射。另一种是传入文件描述符，直接做映射。

一般情况下，Server端会用第一个构造函数；Client会用第二个构造函数。

#### 其他方法

其他所有方法，包括read、write、getFileDescripter等等，都是调用了JNI，在JNI中使用cutils库提供的接口。

#### 总结

MemoryFile的通讯，也即其内部的fd的传递，还是通过Binder，当Client端请求Server端的时候，Server端返回一个fd，Client端会使用第二个构造函数，创建、映射，所以一切的访问就是通过共享内存了。

## 总结

* 
文件描述符都是通过Binder传递的，Server负责创建匿名共享内存、创建文件、完成映射，然后提供Binder服务，供Client获取文件描述符。Client获取文件描述符之后，再进行映射。接下来就可以通过映射出来的地址值访问了。
* 
  文件描述符只对本进程有效，但文件结构体只会有一个，多个文件描述符可以指向同一个文件结构体，文件结构体才真正代表着文件。所以Binder并不能直接传递文件描述符，而是在Client生成一个等价的文件描述符。

  在传递文件描述符时，给Binder的协议是TYPE_FD，则Binder驱动在获取到Server端的文件描述符时，会先查到其对应的文件结构体，然后再查询目标进程中还没有使用的文件描述符，再将文件描述符和文件结构体关联，然后返回给Client端。