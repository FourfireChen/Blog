# View是怎么画出来的（二）——SurfaceFlinger

## 整体定义

* 持有现状态和下一个状态，以及目前的Surface数组，和显示设备数组。以GraphicPlane对象来管理系统的显示设备。
* 有三种线程：

  * Binder线程：用来和用户进程通讯。
  * 控制台线程：用来和内核通讯，内核一旦需要显示设备做响应，则发送信息，通过该线程处理。
  * UI渲染线程：专门负责渲染UI。如果没有渲染任务，则阻塞，有任务则通过gralloc和fb设备，渲染图形。

## 启动

* 
Zygote进程启动System进程，SystemServer执行main方法，会加载android_servers库，调用init1函数，会启动由C++写的服务，其中就调用`SurfaceFlinger.instantiate()`方法。

* 
SurfaceFlinger启动之前，Binder驱动已经启动好了，于是它把自己加入Binder线程池。接下来启动UI渲染线程。再创建一个对象描述显示屏，这个过程中会创建控制台线程，监控控制台事件。
* 
开辟一块匿名共享内存，将显示屏的信息写在上面。

## 管理层次

SurfaceFlinger用一个GraphicPlane描述显示屏，其内部聚合了一个DisplayHardware，用来访问帧缓冲区，DisplayHardware类内部又包含了一个FramebufferNativeWindow，用来真正描述帧缓冲区，它是连接OpenGL库和Android的UI系统的一个桥梁。

## 线程模型

SurfaceFlinger是以UI线程为主要处理逻辑，Binder线程和控制台线程分别用来与用户进程和内核做通讯。UI线程拥有一个消息队列，Binder和控制台接收到请求时，向UI线程发送一个消息，进入其队列。如果没有消息，UI线程阻塞。

## 渲染过程

* 
在SurfaceFlinger中，应用程序窗口以Layer对象描述。SurfaceFlinger持有一个数组，在渲染时会先对Layer的各种属性，判断是否发生了变化。
* 
如果有变化，则调用Layer内部的doTransaction。Layer内部有存有描述状态的对象，根据改变，赋值各个属性。
* 根据变化的属性，设置图形缓冲区的属性。SurefaceFlinger的Layers保存在BufferManager中，并且进行了编号。先标记当前Layer，然后处理其纹理，接下来处理各种区域，如可见区域、透明区域、半透明区域等等，在Layer内部做标记。
* 如果SurfaceFlinger服务在编译时指定了宏USE_COMPOSITION_BYPASS，且当前要渲染的应用程序窗口只有一个，且其可见区域不为空，则跳过合并，直接渲染。在应用程序窗口唯一的时候，SurfaceFlinger为其开辟图形缓冲区是在帧缓冲区上的，也就是直接写入数据，则可通知fb渲染。则渲染完成。
* 
其他的情况，先进行各个图形缓冲区的合成，将各个窗口的各种区域合成到主屏幕上。然后再将主屏幕的内容渲染到帧缓冲区。
* 
上两步的渲染，最终都是通过Gralloc的post方法，这个方法中会判断图形缓冲区是否在帧缓冲区中分配的，如果是，那就是直接写入的，如果不是，需要通过memcpy复制过去。
