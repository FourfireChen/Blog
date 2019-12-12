# 面试准备二——Android

## 四大组件、Fragment

### Broadcast

+  receiver注册：注册在AMS中

    + 静态注册：AndroidManifest。这种注册方式的广播，在应用安装时由PMS完成注册。不跟随Activity，即使应用程序关闭，仍然可以接受到广播。
    + 动态注册：registerReceiver。这种广播跟随注册时的应用，应用关闭后，广播自动移除。

+ 广播分类：重建

    + 普通广播：发送给所有receiver，按注册顺序。
    + 有序广播：按receiver的优先级接收，即priority。队列前的接受者可以随时终止广播的继续发送，也可以修改内容。但可以指定一个final接受者，无论如何都会接收到。
    + 粘性广播：5.0后已经废弃。是指可以先发送，然后再注册也能接受到广播。即广播没有被处理时，会保存在系统中。

+ 本地广播（局部广播）：

    广播注册时有一个exported字段，该字段在有intent-filter时默认为true，没有时为false，这个字段表示能否接收到app外的广播（注意是app不是进程）。

    为了安全考虑，防止接收到外部的广播，形成干扰，可以对需要的广播接收者局部化。有集中措施：

    + exported设置为false；
    + 提供permission，这是一个可以自定义的字段，在接收和发送时都可以指定，用于确认；
    + 发送时可以指定包名，通过intent.setpackage指定。

    **可以通过LocalBroadcastManager发送**

+ onReceive中的Context：

    + 静态注册：ReceiverRestrictedContext，这是一个系统的Context，非Activity
    + 动态注册：Activity
    + LocalBroadcastMamanger：Application

### Activity

+ 启动流程

    从startActivity到新Activity的`onResume()`算启动流程，大概步骤如下：

    1. 原Activity通过Instrumentation，以AIDL的形式，向AMS发送启动新Activity的请求。发送请求的参数包括ApplicationThread（Binder本地对象，供AMS返回调用）、包含启动目的的intent、原Activity的信息、token以及是否要将启动结果返回`startActivityForResult`；

        > Instrumentation用于监控应用和系统之间的交互，应用要请求系统操作都要通过它。

    2. AMS解析Intent的信息，得到新Activity的信息，保存下来。

    3. AMS中会做各种情况的判断，比如是否启动的就是目前栈顶Activity，如果是，是否需要重新启动、启动的是否是同一个APP下的Activity。**此处就是插件化要绕过的主要的地方**，此时会把需要启动的Activity（这个Activity不一定是一开始指定的了）对应的TaskRecord，放到Activity堆栈的顶端，再Activity放到Task的顶部。这个部分和启动方式有关。调用ActivityStack对顶端Activity进行处理。

    4. 处理过程有一个`resumeTopAcitvityInnerLocked`方法很重要，这个方法在整个流程中多次调用，其实个人对这个方法这样写表示一点质疑，因为太臃肿了。这里首先会走第一次，把目标activity放到栈顶，做各种判断之后会通过IPC暂停原Activity，同时埋下一个超时炸弹，然后本进程的这段逻辑会结束。

    5. 值得一提的是从AMS到Activity的通讯，这也是一个IPC，API28对这里做了很大的架构改变。包装成一个**策略**模式？？将各种请求封装成一个对象，交由一个管理者统一管理。这个IPC会将请求统一封装后通过之前传入的ApplicationThread这个Binder对象发送。

    6. ApplicationThread接收到IPC请求之后，转发给ActivityThread。ActivityThread在API28被改为集成ClientTransactionHandler，处理逻辑都在这个类中。

    7. 处理时先调用Transaction对象的preExe，预处理，然后把请求通过Handler发送给mH。

    8. mH会继续交给专门用来处理Transaction的类处理，这里值得一提的是Transaction有一个Callback，会在执行真正的请求之前执行。暂停原Activity的时候没有内容。继续往下执行暂停的请求。

    9. 执行暂停的方法，把ClientTransactionHandler作为回调传进Request的方法中，被调用。

        这里有一个问题，之所以处理逻辑都在`ClientTransactionHandler`之中的一段代码，中间要过一个mH的原因是在过mH之前，ApplicationThread是一个Binder服务端对象，是跑在Binder线程的，要切过来。

    10. 调用ClientTransactionHandler的处理暂停的方法，就是调用ActivityThread里的处理暂停的方法。于是这里会把原Activity给Pause掉。

    11. 在处理完原Activity暂停后，会执行request的postExecute方法，这个方法会重新通过AIDL通知AMS，原Activity已经Pause了。AMS重新调用ActivityStack的方法，这里已经pause成功，于是把一开始埋的炸弹拆除，如果那个炸弹如果超时，会强制activity pause stop 以及有其它的处理，这个不是ANR。由于pause成功，所以继续走下去，会再次来到`resumeTopActivityInnerLocked`。

    12. 做各种判断，准备启动目标activity。这里分两种，一种是进程已经存在的，一种是进程还没创建的：

         - 进程已存在：还是通过封装transaction，把lanch请求当作transaction的callback,，把resume当作request，然后发送，后面就和pause差不多了，通过IPC发送到ActivityThread。

         - 进程不存在：创建进程。埋超时炸弹，然后通过socket和zygote通讯，zygote进程fork一个进程出来然后执行ActivityThread的main方法。main方法会通知会AMS启动已完成。同时，也会创建一个消息循环机制，让当前线程进入消息循环队列。

             然后ActivityThread会再次通过IPC通知AMS，已经启动完成，此时会拆炸弹，然后获取到已经创建的进程对象，然后又有两个逻辑：

             - 重新IPC通知ActivityThread去做一些初始化，如application、Instructation的创建，以及调用它们的生命周期回调。
             - 启动之前已经放置在栈顶的activity，接下来的逻辑就和进程存在时一样了。

    13. 启动Activity的处理被发到ActivityThread后，基本逻辑和pause时一样，处理请求，稍微不同的是这个封装的transaction里面有callback，这个callback是启动activity，这里会创建activity的context、做一些初始化等等，然后调用oncreate。callback先被调，然后就request，这个item会运行onstart和onresume。（onstart的调用方式是cycleTopath）到这里就完成啦。

    14. onStop是什么时候执行的？

         - Activity在Resume之后，会注册一个IdlerHandler，这种Handler在系统空闲时会调用，在这里面会调用ams的activityIdle方法。这里有一个标记位，如果是存在的话，会直接finish掉。
         - 做一些准备之后向ActivityThread发送stop的请求。

+ Activity生命周期

    + 需要注意onRestart->onStart
    + 注意！！onResume的时候，View还没有添进Window中！

+ 跳转Activity时的生命周期

    + 根据上述情况就可以

+ 切换横竖屏时的生命周期：

    在onResume之后会调用onConfigurationChanged，可以在AndroidManifest里面配置不允许切换。View的状态会自动恢复，但数据不会，可以借助ViewModel。

+ Activity堆栈：

    + Task是堆栈的抽象，不指定的情况下，activity以包名为堆栈，同个app位于同个栈，但可以特殊指定，使某些app单独为栈。
    + AMS有前后台栈之分，在显示的为前台栈，其他的是后台栈

+ Activity启动模式

    + standard：每次启动都会创建实例，谁启动的，就在谁的栈中。所以在这个模式下以非Activity的Context启动Activity会抛异常，如果指定new_task位就不会了，但是这个情况在api24-api27，系统自动加了new_task位，导致也可以启动。api27后又修复回来了。
    + singleTop：如果是位于栈顶，则不会重复创建，且onNewIntent会被调用，但onCreate和onStart不会调用。如果不是在栈顶，还是会重新创建。常用防止多次点击。
    + singleTask：完全单例，如果已存在，会弹出其上其它Activity，调用onNewIntent方法。有CLEAR_TOP的效果。常用于主页、登录页
    + singleInstance：完全单例。单独位于一个栈。

+ excludeFromRecent：不会在多任务中显示

+ onConfigurationChanged：在resume之后调用

+ onSavaInstanceState：与targetVersion有关

    + Android3.0之前，在onPause之前执行。
    + Android9.0之前，在onPause之后，在onStop之前执行。
    + Android9.0之后，在onStop之后执行。

+ onRestoreInstanceState：onStart之后 

+ 多个Activity如何退出？

    1. 在Base类中实现KillAll
    2. 广播
    3. 递归，startActivityForResult

+ 进程回收机制

    ​	系统在内存不足或者APP长期在后台运行时，会尝试回收App，这是对进程的回收，整个进程所占用的内存空间都会被回收。

    ​	如果此时重新进入APP，系统会进行恢复，主要是针对Activity的恢复，而恢复的依据是AMS中保存的栈信息，这部分栈信息原本就不属于用户进程，在进程被回收时不会回收这些信息，所以此时打开APP，会依据栈信息重新启动Activity。

    ​	如果任务栈上有多个Activity，恢复时也只会回复第一个Activity，至于如果再按返回键，则会按栈中的信息重新启动上一个Activity。

    ​	同时，其它Activity通过Intent发送过来的数据也是维护在AMS中的，也会有的。

### Service

+ bind和start的区别：
    + 目的不同：start是要启动某一个服务，而bind是要使用某一个服务。如音乐播放器可以start播放音乐，然后在要获取歌曲信息的时候bind的一下获取信息。
    + 生命周期不同：onStartCommand和onBind、onUnBind

+ start流程：
    + Service的启动流程都是在Context里面的，Context的具体实现是ContextImpl。通过AIDL调用AMS的startService。
    + AMS首先通过Binder获取调用的进程记录，然后做各种状态和模式的判断，如果该Service记录已经存在，即已经创建的，这里就意味着重复start，则会IPC通知ActivityThread重复执行onStartCommand；如果不存在，创建一个新的Service记录实例。
    + 接下来会有两种情况，一种是Service指定的进程已存在，另一种是不存在：
        + 不存在：先创建进程，并且将目标Service入队，进程启动后会调用ActivityThread的attach方法，然后重新通知回AMS，此时走的方法和启动Activity时是一样的，优先启动Activity，然后就看看有没有待启动的Service。下面的处理就是进程存在的时候了。
        + 存在：通过ApplicationThread通知ActivityThread，和创建Activity一样，通过mH转进主线程，这时会创建Service实例，初始化ContextImpl、绑定Application等等，然后调用Service的onCreate方法。
    + 在AMS通知ActivityThread创建Service的请求发送之后，就会发送一个通知其Service执行onStartCommand的IPC，然后流程相似。

+ bind流程：
    + IPC通知AMS绑定，而Connection，会连带其他信息包装成一个Binder对象发送过AMS。
    + AMS接受到请求之后，将Connection记录起来ConnectionRecord。接下来判断进程是否存在，如果不存在，和start一样创建进程，然后再次来到这里，通知创建Service，然后再次通知调用onBind。
    + onBind会返回一个Service开发者提供的Binder接口，然后把这个接口对象送回AMS。
    + AMS再次通过IPC，把Binder接口对象发送给每个调用了bindService的Activity。

+ Service生命周期
    + start：onCreate->onStartCommand
    + bind：onCreate->onBind
    + 先start后bind：onCreate->onStartCommand->onBind
    + stop：必须start过
        + 无bind：onDestory
        + 已bind：不能stop
    + unbind：
        + 无start过：onUnbind->onDestory
        + start过：onUnbind

+ Service与Activity通讯
    + bindService中的ServiceConnection，可以获取服务提供者重写onBind返回的Binder对象
    + 通过广播。

+ 前台服务：必须要让用户知道且不允许系统随意杀死的服务，会在状态栏提供通知。

    + 创建：必须提供notification，然后在onStartCommand里调用startForeground，在onDestory里调用stopForeground


### Fragment

+ Fragment生命周期

    + FragmentActivity的onCreate中把FragmentManagerImpl的mCurState设置为CREATED
    + commit：
        + onAttach
        + onCreate：此时Activity已经commit，但视图还没有出来。
    + Activity完成onCreate后，会调用FragmentMananger的dispatchActivityCreated。则会调用fragment的onCreateView->onViewCreated->onActivityCreated。
    + onStart->onResume->onPause->onStop：与Activity对应
    + onDestoryView：对应onCreateView，每次切出都会调用
    + onDestory->onDetach

+ onCreateView和onViewCreate：
    + onCreateView是创建View的，返回的View会被添加到container中
    + onViewCreated在View添加到container中之后的
    + 如果是kotlin直接使用id引用控件，必须要在onViewCreated才可以。

+ Fragment的LayoutInflater默认是host的Context创建的，但有一个onGetLayoutInflater的回调，可以返回指定的LayoutInflater。

+ setRetainInstance：

    可以用于优化，Activity在某些情况下被销毁，如内存不足，此时会销毁其中的fragment的视图，但如果该位为true，则不会销毁fragment实例。

    需要注意的是，重新恢复时不会走onCreate和onDestory

+ Fragment与Activity通讯

    + Activity传给Fragment：setArgment
    + Fragment->Activity：Fragment设置接口，Activity给Fragment传Callback进去

+ Fragment之间通讯：

    + 通过Activity
    + 通过Activity获取FragmentManager，获取另一个Fragment
    + 广播

+ intent-filter：可以有多组，只要中一组就能启动

    + action：Intent中必须存在，和filter中的任何一个一样即可。
    + category：Intent有默认，Intent中的所有category，都必须匹配到filter中的其中一个。
    + data：是URI。其它的和action相似

### ContentProvider

重写增删查改方法。向AMS注册和获取。

## View和动画

+ 事件分发机制：责任链模式

    + 事件分发首先由系统分发到Activity，Activity通过Windows下发到根View，接下来就是Dom树的遍历过程。

    + ViewGroup通过onInterceptTouchEvent判断是否拦截
        + 拦截：自己处理，调用自身View的dispatchTouchEvent
        + 不拦截：调用子View的dispatchTouchEvent。子类如果处理了，自己不处理，如果没处理，再调用自己的onTouchEvent。
    + dispatchTouchEvent：
        + onTouchListener有，调用其onTouch事件，如果返回true，直接返回，不会分发给onTouchEvent
        + onTouch返回false或没有onTouchListener，调用onTouchEvent
    + onTouchEvent：View中会调用onClickListener的onClick方法

+ onTouchEvent：一旦拦截，则整个事件序列都会交给该View处理，一旦忽略DOWN，则整个序列都会跳过。

+ 解决滑动冲突

    + 外部拦截法：重写onInterceptTouchEvent
    + 内部拦截法：重写子View的dispatchTouchEvent方法，首先禁止ViewGroup拦截DOWN事件，然后在需要父View拦截的地方再允许其拦截。

+ Activity、Window、View

    + Activity是逻辑上的组件；Window是视图上的组件，作用是给View提供一切需要的资源，并且提供接口管理View，并不负责绘制；View是视图上的每个小部件，本身负责绘制。
    + Activity通过Window管理View，实际连接Window和View的是ViewRootImpl、WindowManager。
    + 一个Window维护一个viewrootimpl的数组、view数组、params数组。
    + Window的type有三种，0-99是应用程序窗口、1000-1999是子窗口、2000-2999是系统窗口，越大次序越高。
    + viewRootImpl：负责管理View，触发测量、布局、绘制
    + setContentView就是做Window和View的连接。

+ setContentView：Window最根布局是DecorView，包含titleView和ContentParent，setContentView是把id指向的布局加载给ContentParent。完成的是创建，但没有绘制。

+ MeasureSpec：封装的是**父布局**传给子布局的**布局要求！**

    + EXACTLY：父布局希望子布局的大小由size决定
    + AT_MOST：父布局希望子布局的大小最多只能是size
    + UPSPECIFIED：父布局告诉子布局，随意

+ View绘制原理

    + performTraversals会依次调用performMeasure、performLayout、performDraw。
    + performMeasure：会调用measure方法，调用onMeasure，如果是ViewGroup，会对子View所有元素进行measure过程，且必须子View先测量完之后它才能测量。
    + performLayout和Draw类似

+ onMeasure：该回调是父布局为了确定子View的大小，由父布局调用子布局，传递的参数是一种要求，而子布局通过调用setMeasureDimension告诉父布局最终处理的结果。

+ onLayout：同样是父布局为了确定子View的位置，注意的是传入的需要上下左右四个，也就意味着位置和大小都完全确定。

+ 自定义View需要注意的：

    + wrap_content：

        必须重写onMeasure，因为onMeasure有默认实现，而wrap_content传进的MeasureSpec是AT_MOST和父布局的允许最大size，默认实现就是直接照size来的。

    + margin、padding：margin是在父布局onMeasure和onLayout子布局的时候处理，padding需要onDraw时自己处理

+ getWidth和getMeasureWidth：Measure是onMeasure时确定的，但有可能在onLayout时被改了，layout过程传了上下左右四个值，连大小也确定了。

+ 获取View宽高：measure过程和Activity的生命周期是异步的，不能保证在生命周期中可以获得。
    + onWindowFocusChanged：保证View已经显示完毕
    + view.post
    + view.getViewTreeObserver

+ MotionEvent：一般有四种

    + DOWN：按下
    + MOVE：滑动。TouchSlop是最小滑动距离
    + UP：抬起
    + CANCEL：结束，非正常事件

+ onTouch、onTouchEvent、onClick：
    + onTouch如果返回true，后面两个都不会调用
    + onTouchEvent内部调用了onClick

+ 进程类型动画的类型及其原理
    + View动画：res/anim下的xml文件定义，针对View的行为。
    + 帧动画：Drawable，播放图片
    + 属性动画：res/animator下xml定义，也可以代码实现，就是ObjectAnimator之类的工具类。改变的是View的getter、setter，其实就是反射，要求View要改变的属性有提供对应的getter、setter，而且set之后会有实质性影响。如果没有，可以考虑包装一下提供getter、setter。

+ 插值器和估值器：
    + 插值器是刷新速度不同，频率改变
    + 估值器是根据类型不同，如浮点、整型

+ getY不稳定

+ LayoutInflater：通过ID获取到XML文件，以Pull的方式解析，通过ClassLoader加载对应的类。root和attachRoot参数分别指，要加入的父布局和是否加入父布局，分别对应：
    + root==null，attachRoot无意义，不加入父布局，root中设置的参数无用，返回当前解析的布局
    + root!=null，布局中设置的参数有用
        + attachRoot==true，添加到root中，返回root
        + attachRoot==false，不添加到root中，返回当前解析的布局

+ RecyclerView

    - RecylcerView与ListView对比：
        - RecyclerView自带缓存，缓存是四级缓存。ListView缓存必须自己实现
        - RecyclerView对动画做了更多的支持，本身提供了动画支持的API
        - RecyclerView对视图和数据的行为做了更大的解耦，提供了LayoutManager这个管理者单独管理视图
    - RecyclerView绘制：和ViewGroup一样，measure->layout->draw，但RecyclerView把这个过程交给了LayoutManager，所以我们可以自己自定义实现很好看的功能。在这里面会调用子View的onMeasure、onLayout、onDraw。
    - 滑动：滑动也会调用LayoutManager的scrollBy，最重要的机制就在于使用了Recycler，这里便是四级缓存
        - 一级：创建的时候
        - 二级：如果已经出了视线，但没有移除，则在mAttached中，如果移出，则在mCached中，还有一个mChanged，则是notify时记录的需要改变的。
        - 三级：开发者设定的缓存池
        - 四级：大pool，根据viewtype来存，一个type存一条链。

+ Relativelayout和LinearLayout
    + relativelayout比较扁平化，如果层级过多，可以使用其降低层级，减少递归解析View的开销
    + RelativeLayout的onMeasure中对子View的measure进行了两次，横竖各一次，因为需要进行相对化，所以比较慢。但LinearLayout只用了一次，但如果使用了weight，也会先测量一遍所有View，再单独测量weight的。

+ 布局优化：<merge>、<viewstub>
    + merge：主要是解决<include>时的布局过多问题，使用<merge>就直接把子View添进去，不做其他了。
    + viewstub：解决按需加载问题，和include相似，可以指定layout，然后在需要的地方再调用viewstub.inflate方法，才会把内容释放出来。

+ dp、px、dpi：
    + px：真实像素
    + dpi：每英寸的像素点数，也就是密度。勾股定理得到斜边像素数，除以英寸数
    + dp：与密度无关的像素
    + sp：系统设置中的字号改变时，经此设置的大小会发生改变

+ invalidate、postInvalidate、requestLayout：

    + invalidate和postInvalidate只会调用onDraw，request会调用onMeasure、onLayout、onDraw。
    + 主线程用invalidate，子线程用post

## FrameWork

+ Bitmap

+ 消息机制

    + Looper、MessageQueue属于线程，并非每个线程都有，可以自己创建也可以用HandlerThread。native中对应有Looper和nativeMessageQueue。

    + Java的Looper对象持有MessageQueue，MessageQueue的成员变量mPtr是nativeMessageQueue的地址，nativeMessageQueue持有c++的Looper。

    + 创建队列

        线程调用Looper.prepare或prepareMainLooper可在创建该线程对应的Looper和MessageQueue，进而创建NativeMessageQueue和Looper。同时会创建**IO方式**。

        创建eventFd，通过epoll监听。

        > IO方式老版本是使用管道+epoll，C++层的Looper会有写端和读端的fd，新版本改用了eventfd+epoll的IO方式，更高效。

    + 循环过程

        + 调用Looper的loop方法可以开始循环，不断取出MessageQueue的next，也即下一个消息，死循环处理。但next方法可能会阻塞
        + next：开死循环，不停调用nativePollOnce，也会指定睡眠时间，如果没有消息，会睡眠指定时间；醒来时，如果有消息则返回处理，如果没有取出消息，就调用空闲的IdleHandler处理空闲消息。
        + nativePollOnce：调用nativeMessageQueue的pollOnce，进而调用Looper.pollOnce，监听eventfd指向的文件，监听到了，则读出来，清空，这里并不关心读到的数据是什么，只起到通知有消息的作用。至于Message是Java传递的。

    + 发送过程：Handler持有Looper和MessageQueue，自己有handle和send两个方法，handle是处理Looper发来的消息，send是向queue发送消息

        + send：发送到mQueue，插入合适的位置，然后调用nativeWake唤醒，也即通知有消息传入。就是向eventfd写入了一个1。
        + handle：loop中会不断取消息，接收到消息会调用dispatch
            + Message本身有Callback，则执行，返回
            + Hanlder有Callback，执行
                + 返回true，结束
                + 返回false，执行handler本身的handle方法

+ ANR：

    + 前台服务20S，后台服务200S没有启动完成，也即通知ActivityThread启动Service后再次通知AMS完成这个过程。
    + 前台广播10s，后台广播20s，onReceive没有完成。
    + ContentProvider不清楚
    + View事件（包括点击事件）5s内无响应，input系统相关，没研究过，可以想像这个input过程，大概是驱动收集屏幕信息，然后封装，放在某个文件中，input系统监听输入目录或文件下的内容，取出来之后，转发到Activity。在转发前会有埋炸弹，响应了返回了会拆炸弹。

+ IdleHandler：消息循环机制在空闲时会调用，返回true会继续接受消息，返回false则会被删除。

+ Thread添加Looper：调用Looper.prepare或者使用HandlerThread。

+ ThreadLocal：以当前线程为Key，存Value，可使对象线程私有化

+ Zygote：

    + ·init进程加载脚本，启动app_process中的main函数，此时还在native，创建VM实例，调起ZygoteInit类的main方法，进入Java的世界
    + ZygoteInit创建一个Socket，但还没有开始监听。
    + ZygoteInit主动fork System进程
    + 执行runSelectLoop方法，监听Socket，等待AMS通知创建新进程
    + 如果获取到了，fork一个子进程，然后执行，执行的是ActivityThread的main方法。

+ System进程：

    + 由Zygote进程fork出来之后，启动了SystemServer.main方法
    + main方法首先设置各种时区，然后启动各种服务。

+ 新进程的创建：

    + AMS中调用Process.start
    + 先打开Socket，获取到对应文件的inputstream
    + 然后把参数写进去，则Zygote进程会拿到数据
    + Zygote会调用ClassLoader把ActivityThread加载进内存，然后使用反射构造一个实例，并调用其main方法。

+ 为什么要用ClassLoader而不是new：因为这些类没办法静态的写死，比如说Activity，Intent中指定的只是它的包名+名字，也就是路径，要通过这个路径，找到对应的dex文件，动态加载进内存；再比如ActivityThread，也是指定的android.app.ActivityThread类。


## VM

+ Dalvik和JVM的区别：

    + 基于栈和基于寄存器。JVM对移植性要求高，但Android不会。
    + .class和.dex。JVM一个类一个class，Dalvik一个apk只有一个.dex。.class文件记录的信息包括任何有引用的常量、方法签名等，会有比较多的重复信息，但.dex就共享了，但因此也有64K问题

+ ART和Davilk、JIT：

    + Davilk是解释运行，每次运行时逐条解释字节码，翻译成机器指令，效率太低了。
    + JIT：即时编译，以方法为单位，对热点代码编成native，但也是每次APP运行的时候才编译，本身也有性能开销。
    + ART：安装时编译成机器代码，但会使APP占用空间变大，安装时间增加。

+ 内存泄漏：虚拟机的GC策略有关，引用计数和可达性分析，现在多是可达性，只要可达，就不清除，而且一般会分代，一旦一次扫描不被清楚，很可能被认为是常驻的。

    + Context乱传
    + Handler：匿名内部类持有外部类的引用，一旦Handler被其它持有，比如线程，那就很难回收了。可以单独写一个类或者写成静态内部类

    解决：使用弱引用

+ 引用类型：

    + 强：不会随意回收
    + 软：内存足够则不会回收，不足会回收
    + 弱：扫到了就回收
    + 虚：随时可能被回收

+ 虚拟机的发展

    + 5.0之前：JIT
    + 5.0-6.0：AOT，安装时翻译成native
    + 6.0-8.1：JIT+AOT：安装时不编，运行时用JIT，把热点记录下来，空闲时编程native
    + 9.0：似乎使用了共享，对google play上的应用的热点记录文件做共享。

+ 类加载：

    + Java
        + Bootstrap：加载JDK核心类库
        + Extensions
        + Application：应用程序加载
        + 自定义加载
    + 双亲委托模式：先判断Class是否已经加载，如果没有，委托给父类，直到最低端，如果找到，返回，找不到，子类加载。
    + Android：
        + BootClassLoader：加载常用类，只能在lang包内调用。
        + DexClassLoader、PathDexLoader

## 并发

+ AsyncTask：

    onPreExecute在调用线程执行，doInBackground在线程池中执行，返回值是onPostExecute的参数，onUpdateProgress是过程，在主线程调用。

    封装了Handler和线程池。使用的线程池核心线程为1、最大线程为20、有超时制度的线程池，而且进行了封装，保证所有任务串行执行。

+ HandlerThread就是个有Looper的Thread

+ IntentService：继承自Service，是一个抽象类，因为是Service所以优先级高，适合做优先级高的后台任务，任务执行后会自己停止。本质上封装了HandlerThread和Hanlder。

    + 使用方法：重写onHandleIntent。
    + 原理：onStartCommand中会把intent传给onStart，然后封装成消息，通过Hanlder传递给HandlerThread，HanderThread调用onHandleIntent，并结束后调用stopSelf(int startId)，这个方法会检查启动次数是否和startId相等，如相等说明是最后一个任务，自己gg掉。

+ 如何开进程？造成什么影响？

    + AndroidManifest的process字段，四大组建都有
    + 会有多个Application实例，如果自定义要小心重复执行
    + debug时会有多个进程
    + 静态变量失效，进而单例模式失效
    + 通讯
        + Messenger
        + AIDL
        + Binder
        + 文件共享
        + ContentProvider
        + Socket

+ 进程移到前台：

    + Activity正在交互
    + onReceive正在执行
    + Service的回调函数正在执行
    + Service对应的Activity正在交互

+ 线程池：Executor是一个接口，实现类是ThreadPoolExecutor。线程池可以指定核心线程数、最大线程数、线程闲置时间超时时长、工作队列、线程工厂。

    + 线程池中的线程数量未达到核心线程数，创建新线程运行任务
    + 已达到，插入任务队列
    + 若任务队列已满，且不超过最大线程数，则启动非核心线程执行任务。
    + 若超过最大线程数，则拒绝执行，可以注册一个Hanlder供监听调用。

    分类：

    + Fixed：只有核心线程，且线程数固定，线程空闲时不会被回收，而且核心线程没有超时制度。适合快速响应。
    + Cached：只有非核心线程，线程数量不固定，有超时机制，超时回收。适合执行大量耗时较少的任务。用的是Syn队列。
    + Scheduled：核心线程数固定，非核心线程数无限，但闲置下来会被立即回收。适合执行定时任务。
    + Single：只有一个核心线程，确保队列中所有任务都在同一个线程中按顺序执行。

    阻塞队列：

    + 无界队列，大小无限
    + 有界队列（FIFO和优先级）：大小有限
    + syn队列：大小为1，要将一个任务放入，必须要有一个线程在等待接收，如果没有要新建。

+ AIDL：请求是同步的！！！封装了Binder，支持基本类型、String、CharSequence、ArrayList、HashMap（内部元素也要支持）、Parcelable、AIDL的接口

+ Binder

## 其他

+ 保活：提高优先级、在Kill之后重新拉起
    + 进程优先级：简单分为前台（正在交互）、可见（onPause）、服务（startService）、后台（stop）、空，但其实Android系统中将优先级做了细致划分oom_adj，与pid做映射，越大优先级越低
    + 项目需求：锁屏时仍要继续上传定位
    + 锁屏：广播监听，在锁屏时，把原service停止，重启成一个前台服务，这样可以继续传。在解锁时关掉前台服务，重启原服务。这个是出于需求，也可以使用透明Activity。
    + 黑色保活：利用各种广播唤醒service，如网络切换、解锁、拍照等等。
    + 灰色保活：7.1之前，可以启动两个相同ID的前台服务，然后stop掉一个，这样可以避免notification。
    + 白色保活：就前台服务
    + 系统service机制：在onStartCommand返回START_STICKY。系统会在杀死service的时候尝试重启，具体原理是onStartCommand执行结束后，会把返回值返回给AMS，AMS会记录下，在杀死的时候尝试restart，但此时intent是不存在的。
    + onDestory的时候发广播尝试回拉
+ hook：有底层的hook和Java层的hook，Java层的主要就是利用动态代理。
+ 插件化：主要为了解决接入其他端的问题以及64k
    + 启动Activity：主要难点在躲过AMS时的PMS检查
        + Hook IActivityManager：
            + 启动：改掉intent，启动stub，原intent保存下来。
            + 还原：hook mH，给mH增加一个mCallback，会比handleMessage更早执行。
        + Hook Instrumentation：execStartActivity和newActivity
    + 启动Service：借助一个真的Service来调用
    + 资源替换：主要在performLaunchActivity时，会context.setResource，这个是AssetManager的，可以hook替换掉。
+ 适配：res下的图片资源，优先找对应的目录下的，如果找不到，找高分辨率的，再找不到，找最高的。常用对应为1280 * 720—xhdpi，1920 * 1080—xxhdpi
+ apk结构：
    + lib：以CPU结构区分的库代码，主要是SO文件
    + res：开发目录下的res
    + assets：开发时用的assets
    + META-INF：签名文件
    + resources.arsc：记录R.ID和资源文件之间的映射关系
    + classes.dex
    + AndroidManifest.xml
+ apk打包：
    + 打包res
    + 处理.aidl，生成.java
    + .java->.class
    + .class->.dex
    + 整合
    + 签名
+ apk安装：
    + apk复制到/data/app目录下，解压
    + 解析apk里的资源文件
    + 解析AndroidManifest文件，然后在/data/data下建立对应的数据目录
    + 对dex文件做处理，然后保存在dalvik-cache目录下
    + 把AndroidManifest解析出来的四大组件注册在PMS中
    + 发送安装完成广播
+ apk瘦身
    + lib：只保留需要的CPU架构下的SO文件，大部分是arm。一般是ndk的时候在gradle中配置
    + res：
        + 留下一套图，国内大部分机器可以留下xxhdpi，东南亚的可以留下xhdpi
        + 非重要的文件，选择动态加载。
        + webp替换png：webp是google提出来的，甚至支持gif
        + 压缩图片
        + lint工具可以检测出没有使用的图片资源
    + assets：压缩、动态下载、无用删除
+ Serializable和Parcelable：
    + 最大的区别在存储媒介，在跨进程传输时，Serializable存在硬盘中，Parcelable存在内存中，使用了共享内存。
+ 权限：Normal和Dangerous，一种直接在Manifest中声明，一种需要动态申请。
+ LiveData：可观察者，但会遵循Activity的生命周期
    + observe的时候传入lifecycler和observer，会存起来的。在数据来的时候，如果生命周期者在onStart之后和onPause之前，则会调用，因此可以保证不会在Activity Destory的时候调用，不崩，也不会内存泄漏。
    + LiveData有一个version变量，lifecycler有一个version，每次分发数据的时候要比较，如果lifecycler的版本比较低，才会分发，这样保证了数据先到而Activity后启动的时候，还可以获得数据。
+ ViewModel：目的是保证在配置改变时，Activity的重建不会影响到View的状态。
    + 创建时要传入Fragment或FragmentActivity。通过Provider反射创建ViewModel，然后存入对应的Fragment或FragmentActivity的ViewModelStore中。
    + ViewModelStore是每个FragmentActivity或Fragment都有的，其原理是为Fragment或Activity添加一个HolderFragment，并且调用setRetainInstance(true)，这样子就完美啦。
+ MVC、MVP、MVVM

## 开源库

+ okhttp
+ retrofit
+ butterknife
