# View是怎么画出来的（四）——应用程序窗口

## 关系概述

* 
Android中，四大组件与UI相关的就只有Activity，Activity在启动之后，会通过一个类型为Session的Binder对象与WMS维持链接，Activity窗口在WMS中对应的是WindowState，描述窗口信息。
* 
Android应用进程会通过WMS，请求SurfaceFlinger创建Layer，返回SurfaceLayer Binder对象。则WMS和应用进程中都会持有SurfaceLayer代理对象，都可以和Layer通讯。
* 
Activity和ContextImpl是互相持有的，互相可以调用对方。
* 
Activity的成员变量有一个mWindow，类型为Window，用来描述一个应用程序窗口，其真正实现类是PhoneWindow，每个Activity都有一个对应的Window。同时Window内也持有Activity作为Callback的实例，可以调用Activity实现的某些接口方法。
* 
Window和Activity都持有同一个WindowManager代理。
* 
PhoneWindow中持有mDecor和mContentParent，分别指的是DecorView和Content。
* 
DecorView的绘制是由ViewRoot来管理的，Activity在启动的过程中，会给自己创建一个ViewRoot对象，也会将PhoneWindow中的mDecor保存在ViewRoot的mView中。所以ViewRoot可以访问DecorView。ViewRoot用来管理绘制、接收IO等事件。
* 
ViewRoot继承了Handler，所以它是可以接收消息的。如接收到InputMnager的输入消息、重绘。
* ViewRoot持有了mView，也持有WindowManager.LayoutParams，描述的是Activity的UI布局信息，实时上是DecorView的。ViewRoot、DecorView、LayoutParams都保存在WindowManagerImpl中，分别在三个列表中，通过索引号对应一个Activity下的三者。一个应用程序有几个Activity，这三个列表的大小就是多少。
* ViewRoot有一个类型为Surface的成员变量mSurface，这是JAVA层的Surface，其引用了Native层的Surface，这个就是和SurfaceFlinger相关的了，这是SurfaceFlinger机制中，应用程序窗口在应用进程端的描述，一个Surface会对应一个SurfaceControl，用来设置窗口属性，如大小、位置等，但ViewRoot没有SurfaceControl，因为它不需要设置属性，只需要填充内容，即只需要填充纹理。
* Java层的Surface持有一个Canvas，通过它可以访问到图形缓冲区，绘制纹理。而Surface的属性由WMS来管理。
* ViewRoot持有和WMS通讯的Session，当Activity第一次要绘制时，ViewRoot会通过Session请求WMS，WMS请求SurfaceFlinger创建Layer，返回SurfaceLayer，然后封装成Java层的Surface，返回给ViewRoot。
* WMS中，每个Activity对应一个WindowState，其内部持有一个Session和Surface，Session是SufaceSession（每个应用进程对应的多个WindowState所对应的SurfaceSession是一样的），持有mClient，是Native层SurfaceComposerClient对象的地址，用来和SurfaceFlinger通讯。

## 运行上下文

应用程序窗口在运行过程中需要一些资源，这些资源共同构成了上下文，Activity本身就是一个上下文，它继承了ContextThemeWrapper、ContextWrapper，这是一个装饰器模式，真正的实现是ContextWrapper内部持有的ContextImpl，ContextImpl也会通过OuterContext持有Activity。

* 创建ContextImpl，然后跟新创建的Activity关联，调用的是attach方法。
* 此时创建一个PhoneWindow对象，然后保存为mWindow。

## Window

PhoneWindow在构造方法中，会根据传进来的Context创建一个LayoutInflater，保存在成员变量中。

## View的创建

每一个Window都有一个ViewRoot，ViewRoot相当于MVC中的Controller，它负责：

* 为Ｗindow创建Surface对象。
* 调用WMS来管理窗口。
* 管理、布局、渲染UI。

这个过程的起点在开发者重写的onCreate中的，setContentView。

### setContentView

这个方法就是给DecorView的content部分填充内容。

* 调用的是PhoneWindow的setContentView
* 将DecorView加载好，包括title和content。
* 然后把传进来的LayoutId，通过Inflater加载进content部分。

## Resume

这个过程是激活的过程，目的是创建ViewRoot，和DecorView关联起来，然后显示。

* 调用Activity的onResume。
* 获取PhoneWindow以及DecorView、LayoutParams，调用WindowManagerImpl的addView，创建ViewRoot，然后将三者关联。并将params和父布局params设置到ViewRoot中。
* 接下来ViewRoot会调用requestLayout对应用程序窗口UI做第一次布局。
* 调用IWindowSession的add方法，通知WMS创建一个WindowState。这里会把W对象传过去，以便WMS可以和应用进程通讯。

## WMS

应用进程端，每个Activity都持有一个W对象，实现了IWindow窗口，这个对象的作用类似ApplicationThread，也是一个Binder本地对象，ApplicationThread用来和AMS通讯，而W对象用来和WMS通讯。

* 
  Activity持有Window，Window持有IWindowSession，可以用其和WMS通讯，作用在于：

  * 调用add方法，将W对象传递过去，请求WMS创建一个WindowState对象。
  * 调用remove方法，销毁WindowState对象。
  * 调用relayout方法，请求WMS对UI进行布局。
* 
  WMS持有W对象，可以用来：

  * Activity组件大小、可见性、焦点性发生变化时，系统会通知WMS，WMS可以通过W通知Activity。
* 
  AMS和WMS之间有连接，通过一个AppWindowToken来描述。

  Activity在启动时，AMS会为其创建一个ActivityRecord对象，并以此请求WMS创建一个AppWindowToken对象。

  AMS在启动完成之后，会把ActivityRecord返回给应用进程，应用进程在调用add方法时，会把W方法和ActivityRecord一起传给WMS。

  WMS会创建WindowState，此时也会创建和SurfaceSession的连接，然后根据ActivityRecord找到AppWindowToken，把它保存在WindowState中。

## Surface

Java层的绘图表面，由两个Surface描述，一个是应用进程端中Window中保存的Surface，一个是WMS中WindowState中保存的Surface，这两个Surface都持有C++的Surface的地址值，指向的是同一个Native层的Surface。

几个Surface的创建过程：

* 应用进程请求WMS创建Surface。
* WMS请求SurfaceFlinger创建Layer，返回Surface。
* WMS获得Surface后保存在WindowState中，再将其返回给应用进程。
* 应用进程获得Surface后保存在Window中。

### requestLayout

这个方法是在Resume过程中调用的，是ViewRoot的成员方法。也是UI绘制的起点。

* 检查是否是UI线程（主线程）。
* 调用`ViewRoot.scheduleTraversals`。发送一个DO_TRAVERSAL消息，因为ViewRoot本身是一个Handler，所以会触发其本身的handleMessage方法，会调用performTraversals。

* 首先调用mDecor的Measure，触发整棵View树的测量。
* 如果当前处理的窗口UI是第一次被处理，或大小、可见性、布局参数发生变化，则会调用relayoutWindow来请求WMS重新布局**系统中的窗口**（注意不是View，是窗口）。
* 调用mDecor的layout流程，对当前窗口的UI重新布局。
* draw

### relayoutWindow

刚刚创建的Java层Surface是没有对应的Native层Surface的，在调用到WMS的relayoutWindow时，才会创建C++的Surface。

真正的Surface，需要有SurfaceControl、Canvas、Name，这些属性在Surface的构造方法中会初始化，此时会调用JNI方法，在Native层创建SurfaceControl，以及请求SurfaceFlinger创建Layer，返回Surface。

## 真正绘制

应用程序向图形缓冲区填充数据，然后通知SurfaceFlinger绘制。但应用程序进程一般不会直接向图形缓冲区写数据，而是通过一些API来绘制，比如OpenGL、Skia。Skia中，所有的绘制都是在画布Canvas上操作的。所以Android系统将图形缓冲区包装成一个Canvas。

整个绘制流程是常说的Measure、Layout、Draw。

### Measure

先传到DecorView的measure方法，参数是最大宽、高，对顶层视图来说，也就是屏幕宽高。接下来会调用自己的onMeasure方法，传进的也是对宽高的期望值。往后就是一个递归过程了。此处以FrameLayout为例。

* 触发子View测量。
* 子View测量完之后，则可以知道子类的宽高，即mMeasuredWidth和mMeasuredHeight。
* 根据子类的宽高，确定自己的宽高，然后设置自己的宽高，设置的是mMeasuredWidth和mMeasuredHeight。

### Layout

会调用到View的layout方法。

* 首先根据传进来的参数，调用setFrame，设置当前View的位置和大小。
* 调用onLayout，重新布局子View。
* 布局方式由自己决定，传给子View上下左右，调用的也是子View的layout方法，所以这是一个递归过程。

### Draw

* 首先获取mSurface，然后做各种是否需要动画处理的判断。
* 判断是否直接以OpenGL来绘制，这样可以直接使用GPU来绘制，如果是的话，获取OpenGL相关的画布，然后调用ViewRoot的draw，触发整个View树的绘制，把Canvas传进去。这种选项需要特殊配置。
* 如果不是以OpenGL，进过一系列计算后，请求Surface返回一块画布，然后传给ViewRoot触发绘制。
* 绘制完成之后，再调用Surface的方法

unlockCanvasAndPost
，把Canvas上的东西推给SurfaceFlinger，交给其绘制。

正常情况下都不会使用OpenGL渲染，下文的流程是非OpenGL流程。

#### 创建画布

调用的是Surface.lockCanvas获取Canvas。

* 调用Native层的lockCanvasNative。
* 找到C++ Surface，调用lock方法，获取图形缓冲区（这个过程是上一篇文章写的关于应用进程图形缓冲区的获取，dequeueBuffer），封装成SurfaceInfo对象，图形缓冲区地址保存在SurfaceInfo内部。
* 找到Java层Surface的成员变量Canvas，把图形缓冲区地址保存到其内部。

#### DecorView.draw

终究也是调用View的draw方法

* 首先会对layout的结果做反应，移动Canvas。
* 调用onDraw。
* 裁剪Canvas，调用子View的draw，把裁剪好的Canvas传进去。

#### unlockCanvasAndPost

会调用JNI方法。请求SurfaceFlinger渲染这块图形缓冲区，queueBuffer。

## 问题

为什么在应用端使用Skia API来画Canvas，然后再推到SurfaceFlinger用OpenGL真正渲染？
