# View是怎么画出来的（三）——应用进程和SurfaceFlinger

## 关系

* 
  应用进程和SurfaceFlinger的通讯是通过Binder来实现的。每一个Application都和SurfaceFlinger有一个连接，这个连接通过一个类型为Client的Binder对象来描述，应用程序和SurfaceFlinger建立连接之后，会得到一个Client的代理对象，则可以使用这个代理对象了。
* 
  应用进程和SurfaceFlinger通讯的时候，需要将UI元数据（元数据是指各种属性）发送给SufaceFlinger，这个数据量不是一个小数据，尽管Binder只有一次内存复制，但这个效率还是不够高，所以Android系统中的实现使用的是共享内存，这种共享内存的机制，是Android独有的Ashmem匿名共享内存。
* 
  应用进程和Client之间有一块共享内存，这块共享内存由类型为SharedClient的对象描述，其中存着多个SharedBufferStack，每一个SharedBufferStack对应SurfacFlinger中的一个Surface，也对应这应用程序中一个窗口。

> 为什么需要Stack？
>
> 一般UI渲染都会使用“双缓冲”技术，最顶层的显示器这一层是不能够直接绘制的，因为在其上直接绘制会导致画面撕裂、卡顿、模糊等等各种问题，所以多了一层缓冲区，绘制都在缓冲区上，缓冲区和显示屏只进行复制数据的操作，以此保证每一帧的完整性。
>
> 有了Stack，则可以使用多缓冲了，在Android4.1中，据说3缓冲。

* 
  SharedBufferStack的主要内容是一个链表，其元素Buffer保存的是UI元数据，每个元素对应一个GraphicBuffer，这是真正的UI数据。

  * **应用进程**要更新一个Surface时，找到其对应的SharedBufferStack，取出一个**空闲**的Buffer，然后请求SufaceFlinger为这个Buffer对应的编号分配一个图形缓冲区GraphicBuffer。
  * 接着把其返回给应用进程，应用进程向其写入UI数据。
  * 写完之后，其对应的Buffer插入到SharedBufferStack的**非空闲**缓冲区中。
  * 应用进程通知SurfaceFlinger服务去绘制那些Buffer对应的GraphicBuffer。

这整个过程传递的只需要是Buffer的编号，因为GraphicBuffer存在SufaceFlinger中，根据编号是可以查到的。
* 
应用进程看的是SharedBufferStack的空闲区，而SufaceFlinger用的是其非空闲区。所以为了方便，Android系统分别使用SharedBufferServer和SharedBufferClient来描述非空闲列表和空闲列表。
* 
GraphicBuffer内部有一块用来保存UI数据的缓冲区buffer_handle_t，它就是图形缓冲区，是分配于系统帧缓冲区或匿名共享内存。

## 连接过程

这个过程是以Binder为基础搭建的。

* 应用程序首先通过Binder机制获得SufaceFlinger的代理对象。
* 每个应用进程与SurfaceFlinger之间有且仅有一个SurfaceClient，以单例实现，所以第一个获取的时候，需要创建，此时会通过SurfaceFlinger代理对象，请求创建一个匿名共享内存，然后返回给应用进程，并在用户进程强转为一个SurfaceClient对象。

## Surface

Suface是一张纸，应用进程负责在上面画东西，而SurfaceFlinger负责把上面的东西拓印到显示屏上。

* 在SurfaceFlinger中，Surface以Layer来描述。

  Layer是LayerBaseClient、LayerBase的子类。

  Layer持有SharedBufferServer，即SharedBufferStack的待渲染列表。

  Layer还持有一个Binder本地对象SurfaceLayer（类似ApplicationThread），SurfaceFlinger就是用其与应用进程通讯。

* 在应用进程中，每一个Surface是由一个SurfaceControl来创建的，SurfaceControl持有SurfaceLayer的代理对象，也持有一个Surface、SharedBufferClient（SurfaceClient）。

  Surface类继承EGLNativeBase、ANativeWindow，而ANativeWindow是OpenGL中的本地窗口概念，Android通过OpenGL绘制（注意和渲染的区别）UI，所以Surface也是用来描述一个本地对象的。

总：对于一个绘图表面，即上文说的一张纸，应用进程端以Surface描述，SurfaceFlinger端以Layer描述，分别持有SharedBufferClient和SharedBufferServer，用来描述UI元数据缓冲栈。

### 创建

应用进程在创建一块Surface时，在应用进程端会创建SurfaceControl对象、一个Surface、一个SharedBufferClient，在SurfaceFlinger端创建一个Layer、一个SurfaceLayer、一个SharedBufferServer。

* 通过Binder获取SurfaceFlinger的代理对象。
* 通过SurfaceFlinger的代理对象，请求SurfaceFlinger创建一个Surface，返回Surface的各种信息和Surface的代理对象，这个代理对象在服务端的引用是SurfaceLayer。

  * SurfaceFlinger端：

    * 正常情况下，会创建一个新的Layer对象（有特殊情况如对原Surface进行模糊，则需要获取之前的Surface）。然后在其内部创建一个SurfaceLayer。
    * 将创建好的Layer和其对应的应用进程对应的Client对象关联，保存在Client内部的mLayers中。
    * 将Layer保存到SurfaceFlinger的mCurrentState内部的layersSortedByZ中。
    * 返回
* 把Surface的各种信息和代理对象封装成SurfaceControl对象。

接下来应用进程要使用Surface时会调用SurfaceControl的getSurface。

* 根据之前已经返回过来的Surface的信息，创建一个Surface
* Surface构造过程中，会创建SharedClient，以及其他的包括SurfaceLayer的代理对象、各种属性等等。
* 调用初始化方法：

  * 第一部分：设置OpenGL本地窗口的各种回调，比如对空闲缓冲区的获取和对待渲染队列的插入。
  * 第二部分：设置UI元数据，如旋转方向、纹理等等，以及SharedBufferClient等等。
* 通知SurfaceFlinger获取、设置ShredBUfferServer。

## 渲染

* 初始化OpenGL的默认显示屏。
* 将Surface转换为OpenGL的Surface—EGLSurface，并获取EGLContext作为绘图上下文。
* 从UI元数据缓冲栈中找到一个空闲的元数据缓冲区，将其编号发送给SurfaceFlinger。
* SurfaceFlinger分配一个图形缓冲区，SurfaceFlinger调用进程对应的Layer，进而调用Gralloc模块，直接在帧缓冲区或在内存分配图形缓冲区，这个图形缓冲区以GraphicBuffer描述，这是一块匿名共享内存。这个过程将涉及到gralloc模块的打开、分配等等操作。
* 分配完成之后，将图形缓冲区和编号做关联，则以后都可以通过编号来通讯，极大减少通讯数据量。
* 返回给应用进程，应用进程将其映射到本进程地址空间，并且UI元数据和图形缓冲区将会做关联，对图形缓冲区做绘制。
* 将对应的UI元数据插入待渲染队列。
* 通知SurfaceFlinger渲染，SurfaceFlinger的UI渲染线程原本会阻塞，此时会被唤醒，将待渲染队列拿出来，取出图形缓冲区，此时可能有多个图形缓冲区，先进行合并，然后渲染到fb设备的帧缓冲区上。
