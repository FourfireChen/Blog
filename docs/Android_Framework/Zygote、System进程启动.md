# Zygote、System进程启动

## 概述

​ Zygote，翻译为中文是“受精卵”，它是Android系统中，所有Java进程的父进程，负责创建新进程。System进程，是Android系统中系统服务运行所在的进程，它负责创建、管理所有的系统服务，包括AMS、PackageManagerServer等等十分重要的服务，都运行在System进程中。

​ Zygote和System进程的生命周期伴随整个系统，在系统启动时就会启动这两个进程，而直到系统关闭，它们才会被杀死。

​ 值得一提的是，在Android5.0之后，Zygote进程在系统中一共存在两个，这主要是为了适应新增加的64位app而设计的，它们两个的主要功能其实是一致的。

​ And，System进程其实也是Zygote进程的子进程，它们启动的顺序大概可以描述为，Linux系统的初始化进程`<init>`通过读取脚本，创建Zygote，Zygote创建完成之后，先启动了System进程，然后自己再等待其他创建请求。

​ 本文以API28的源码为基础进行分析。

## Zygote启动过程

### 总述

1. init进程加载脚本，启动app_process文件中的main函数

1. 创建虚拟机实例，进入ZygoteInit类的main方法中

1. 创建一个Socket

1. fork出System进程

1. 自己进入死循环中，不断从第3步创建的Socket中获取是否有创建进程的请求，并进行处理。

### Step1: app_main.main

**frameworks/base/cmds/app_process/app_main.cpp**
```java
int main(int argc, char* const argv[]){
    ...
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    ...
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        }
        else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        }
        else if (strcmp(arg, "--application") == 0) {
            application = true;
        }
        else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        }
        else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        }
        else {
            --i;
            break;
        }
    }
    ...
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    }
    else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    }
    ...
}
```
走这个方法的目的是解析启动脚本中传进来的参数并真正调用方法启动进程。

1. 创建一个AppRuntime对象，是AndroidRuntime的子类，它的定义就在app_main文件中。

1. 做一些解析工作，解析从启动脚本传过来的参数，这里要启动的是zygote进程，所以zygote变量为true。

1. 调用runtime的start方法，runtime就是第一步创建的AppRuntime对象，但是它没有重写start方法，start方法是在它的父类AndroidRuntime中的。注意：传入的参数是ZygoteInit类的全名以及要传给这个类的参数，以及最后一个判断启动的是否是zygote进程的bool值。

### Step2: AndroidRuntime.start

**frameworks/base/core/jni/AndroidRuntime.cpp**
```java
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote){
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    ...
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
    。。。char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    ...
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    ...
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;)V");
    ...
    env->CallStaticVoidMethod(startClass, startMeth, strArray);
    ...
```

走这个方法的目的是从c++的世界转入Java的世界。

1. 初始化JNI

1. 创建并开启一个虚拟机实例

1. 给这个虚拟机注册JNI方法

1. 各种解析之后，调用`env->CallStaticVoidMethod`方法，从而调起Java的方法，传入的三个参数分别是要调用的Java类、要调用的方法、传给方法的参数，这里要调用的Java类正是从上一步传进来ZygoteInit类的全称解析出来的，而`startMeth`参数是main方法，所以这里调用起来的是ZygoteInit的main方法。

**宣布进入Java的世界！**

### Step3: ZygoteInit.main

```java
public static void main(String argv[]) {
    ZygoteServer zygoteServer = new ZygoteServer();
    ...
    boolean startSystemServer = false;
    String socketName = "zygote";
    String abiList = null;
    boolean enableLazyPreload = false;
    for (int i = 1; i < argv.length; i++) {
        if ("start-system-server".equals(argv[i])) {
            startSystemServer = true;
        }
        else if ("--enable-lazy-preload".equals(argv[i])) {
            enableLazyPreload = true;
        }
        else if (argv[i].startsWith(ABI_LIST_ARG)) {
            abiList = argv[i].substring(ABI_LIST_ARG.length());
        }
        else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
            socketName = argv[i].substring(SOCKET_NAME_ARG.length());
        }
        else {
            throw new RuntimeException("Unknown command line argument: " + argv[i]);
        }
    }
    ...
    zygoteServer.registerServerSocketFromEnv(socketName);
    ...
    if (startSystemServer) {
        Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
        // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
        // child (system_server) process.if (r != null) {
            r.run();
            return;
        }
    }
    ...
    caller = zygoteServer.runSelectLoop(abiList);
    ...
    if (caller != null) {
        caller.run();
    }
```

1. 创建ZygoteServer，然后做各种标记位的判断，比如`startSystemServer`判断是否要启动System进程，这里为true。
2. 调用`zygoteServer.registerServerSocketFromEnv`(详见Step3.1)，创建一个Server端Socket，用来等待AMS请求，以创建应用程序进程。但这时候只是创建，还没开始等待。
3. `forkSystemServer`(详见Step3.2) 创建一个新的子进程，这个新的子进程就是System进程，这里讨论的是Zygote的创建，这个方法会返回null，具体原因见Step3.2
4. 调用`runSelectLoop`方法，在这个方法中无限等待请求。

#### Step3.1: ZygoteServer.registerServerSocketFromEnv

```java
void registerServerSocketFromEnv(String socketName) {
    if (mServerSocket == null) {
        int fileDesc;
        ...
        String env = System.getenv(fullSocketName);
        fileDesc = Integer.parseInt(env);
        ...
        FileDescriptor fd = new FileDescriptor();
        ...
        mServerSocket = new LocalServerSocket(fd);
        ...
    }
```

这是Step3的第2步，创建并注册Socket。

通过socket的名字创建文件操作符，然后再通过文件操作符创建一个socket，其实这个socket在操作系统中的表现形式就是一个文件（这是关于Linux系统的知识，这里不详述）这个socket就存在成员变量`mServerSocket`中。

#### Step3.2: ZygoteInit.forkSystemServer

```java
private static Runnable forkSystemServer(String abiList, String socketName,ZygoteServer zygoteServer) {
    ...
    String args[] = {
    "--setuid=1000","--setgid=1000","--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1024,1032,1065,3001,3002,3003,3006,3007,3009,3010","--capabilities=" + capabilities + "," + capabilities,"--nice-name=system_server","--runtime-args","--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,"com.android.server.SystemServer",}
    ;
    ...
    pid = Zygote.forkSystemServer(parsedArgs.uid, parsedArgs.gid,parsedArgs.gids,parsedArgs.runtimeFlags,null,parsedArgs.permittedCapabilities,parsedArgs.effectiveCapabilities);
    ...
    //* For child process /*
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }
    return null;
}
```
这是Step3的第2步，它的目的是创建System进程

1. 首先是置顶了一些参数，这些参数主要是针对即将创建的子进程，即System进程的，可以看到这里为System进程设定了其UID、GID等等信息，而且指定了它的执行类是`com.android.server.SystemServer`。
2. 调用`Zygote.forkSystemServer`方法，这个方法会调用操作系统提供的fork方法，fork方法是Linux系统中创建一个新进程的方式，是一个非常特殊的方法，父进程执行fork方法之后，会出现两个几乎完全一样的进程，即原先的父进程和新建的子进程，同时返回fork方法，而且继续执行的代码是一致的，但是不同的是，fork的返回值不相同。在父进程中，会返回子进程的pid（ProcessID），而子进程中会返回0。
3. 然后看接下来的代码，if语句判断pid是否为0，所以if语句块中的代码是子进程会执行的，而对于父进程，pid不为零，所以下面会返回null，这也就解释了Step3中为什么是返回null。读者看到这里可以重新返回去看看step3就会明白。（这种写法是非常典型的fork子进程之后的写法）
4. 这里重新理一理Step3最后的步骤，在父进程，也就是Zygote进程中，`forkSystemServer`方法返回了null，于是会执行`runSelectLoop`方法，而子进程，也就是System进程，返回的是一个runnable，是由`forkSystemServer`方法中调用的`handleSystemServerProcess`方法返回的，它嵌套返回到main方法中后，会执行run方法，然后main方法就返回掉了，因为这是System进程了。

#### Step3.3: ZygoteServer.runSelectLoop

```java
Runnable runSelectLoop(String abiList) {
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
    fds.add(mServerSocket.getFileDescriptor());
    ...
    while (true) {
        StructPollfd[] pollFds = new StructPollfd[fds.size()];
        for (int i = 0;
        i < pollFds.length;
        ++i) {
            pollFds[i] = new StructPollfd();
            pollFds[i].fd = fds.get(i);
            pollFds[i].events = (short) POLLIN;
        }
        ...
        Os.poll(pollFds, -1);
        ...
        for (int i = pollFds.length - 1;
        i >= 0;
        --i) {
            if ((pollFds[i].revents & POLLIN) == 0) {
                continue;
            }
            if (i == 0) {
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            }
            else {
                try {
                    ZygoteConnection connection = peers.get(i);
                    final Runnable command = connection.processOneCommand(this);
                    if (mIsForkChild) {
                        // We're in the child. We should always have a command to run at this// stage if processOneCommand hasn't called "exec".
                        if (command == null) {
                            throw new IllegalStateException("command == null");
                        }
                        return command;
                    }
                    else {
                        // We're in the server - we should never have any commands to run.
                        if (command != null) {throw new IllegalStateException("ccdfc54e82fe40e0974d3f0203271ff7");}
                        // We don't know whether the remote side of the socket was closed or
                        // not until we attempt to read from it from processOneCommand. This shows up as
                        // a regular POLLIN event in our regular processing loop.
                        if (connection.isClosedByPeer()) {
                            connection.closeSocket();
                            peers.remove(i);
                            fds.remove(i);
                        }
                    }
                }
                ...
            }
        }
    }
}

```
前面说到这里是Zygote进程死循环等待的地方，是Zygote进程fork出System进程之后，自己等待别的请求的地方。

1. 首先声明了两个数组，一个是文件描述符数组，第一个元素放的就是`mServerSocket`。另一个数组是`ZygoteConnection`数组，从类名就能猜出（当然事实也确实是这样滴）这个类描述的是一个和Zygote进程的连接，也就是说当Zygote接收到请求之后，会将它封装成一个ZygoteConnection对象。
2. 开个死循环，调用方法`Os.poll`，处理轮询状态。poll也是Linux提供的系统调用，用于获知指定的Socket是否有事件到达。可以看到这里创建了一个`StructPollfd`数组，StructPollfd类是专门与Poll调用配合的一个类，用来描述poll调用的目的和结果，它既包含了一个描述poll监听目的的成员变量，也包含了一个描述已发生事件的成员变量。Poll调用根据传入的StructPollfd，指定监听的socket，指定监听的事件类型，如果没有事件到达，会阻塞在poll方法中，如果有事件到达，则将发生的事件写入StructPollfd对象中，然后返回。
3. 可以看到，一开始装入的`pollfd`是`mServerSocket`，然后进入poll调用，一旦有事件到达，`poll`方法将跳出，进入下面的for循环，这时如果i=0，就是AMS请求与Zygote连接（注意这里并非请求创建进程），AMS一般在第一次请求创建进程时会先请求连接，之后如果没有关闭这个链接，则再请求创建之时不用请求连接。Zygote接收到连接请求，则将请求包装后，放入peers中，然后又回到Poll了。注意这里的连接，即`ZygoteConnection`对象，描述的是和AMS的连接，并不会包括AMS请求创建进程所传递的任何参数。
4. 等到AMS再次请求创建时，取出peers中对应的连接，处理、调用`processOneCommand`方法。这个地方Poll的作用就是在于监听到socket有数据传入了，这些数据才是AMS请求创建进程所传递的数据，而读出这些数据的地方就在`processOneCommand`方法中。
5. 往下又是一个非常典型的fork之后的写法，如果是子进程，就把processOneCommand的结果，即一个Runnable返回出去，在前面提到的ZygoteInit的main方法中会调用它run。父进程会判断是否和AMS已经断开了，通过方法`connection.isClosedByPeer()`判断的。如果已经断开，则把连接删除掉，如果没有断开，就继续在poll等待AMS请求创建进程。

#### Step3.4：ZygoteConnection.processOneCommand

```java
Runnable processOneCommand(ZygoteServer zygoteServer) {
    ...
    //从socket中读出要创建的进程的参数args = readArgumentList();
    ...
    parsedArgs = new Arguments(args);
    ...
    pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids, parsedArgs.runtimeFlags, rlimits, parsedArgs.mountExternal,parsedArgs.seInfo, parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.startChildZygote, parsedArgs.instructionSet, parsedArgs.appDataDir);
    ...
    if (pid == 0) {
        // 子进程zygoteServer.setForkChild();
        // 子进程中要关掉ServerSocketzygoteServer.closeServerSocket();
        ...
        return handleChildProc(parsedArgs, descriptors, childPipeFd, parsedArgs.startChildZygote);
    }
    else {
        // 父进程，即ZygotehandleParentProc(pid, descriptors, serverPipeFd);
        return null;
    }
    ...
}
```

这个方法要处理创建进程的请求，在这里会fork出子进程了。

1. 调用`readArgumentList`方法，从`mServerSocket`中读出AMS放进去的请求参数，并封装一下
2. 调用`Zygote.forkAndSpecialize`方法fork出子进程
3. 父进程和子进程分开处理。这个地方涉及到的主要是应用程序进程的启动过程，这里不详述。

***到这里Zygote就启动完成啦，在死循环里面等着。***

## System启动

### Step1: ZygoteInit.handleSystemServerProcess

```java
private static Runnable handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {
    ...
    if (parsedArgs.invokeWith != null) {
        ...
    }
    else {
        ClassLoader cl = null;
        if (systemServerClasspath != null) {
            cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);
            Thread.currentThread().setContextClassLoader(cl);
        }
        // Pass the remaining arguments to SystemServer.
        return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }
    ...
```

1. 这个方法在ZygoteInit里调用的，前面有说到。而且它的调用顺序是在Zygote的死循环开始之前的。
1. 它的参数是前面设置好的一个参数，而且是指向子进程的，也就是system进程。其中`invokeWith`是没有设置的，应该是为了检测进程内存泄露等等Debug用途时才会有值，这一点如果有大大知道，望指点一二。
1. 接下来调用`ZygoteInit.zygoteInit`

### Step2: ZygoteInit.zygoteInit

```java
public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
    ...
    ZygoteInit.nativeZygoteInit();
    return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
    ...
}
```

1. 
    `nativeZygoteInit`在System进程中启动了一个Binder线程池。这里不详述，关于Binder的知识后面会写文章。

1. 调用了`RuntimeInit.applicationInit`

### Step3: RuntimeInit.applicationInit

```java
protected static Runnable applicationInit(int targetSdkVersion, String[] argv,ClassLoader classLoader) {
    ...
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```

继续调用`findStaticMain`

### Step4: RuntimeInit.findStaticMain

```java
protected static Runnable findStaticMain(String className, String[] argv,ClassLoader classLoader) {
    Class<?> cl;
    ...
    cl = Class.forName(className, true, classLoader);
    ...
    Method m;
    ...
    m = cl.getMethod("main", new Class[] { String[].class });
    ...
    return new MethodAndArgsCaller(m, argv);
}
```

这里调用了className指向的类的main方法，也就是`SystemServer类的main方法。包装了一下返回了一个`MethodAndArgsCaller`对象，这个类实现了Runnable接口，于是这个对象会一直返回一直返回到ZygoteInit方法中。

值得注意的是，这里返回到`ZygoteInit.main`方法中的进程，不是前面的Zygote进程，而是System进程，是由Zygote进程fork出来的。我们重新看看main方法中的那一段代码。

```java
Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
// { @code r == null} in the parent (zygote) process, and { @code r != null} in the
// child (system_server) process.
if (r != null) {
    r.run();
    return;
}
```

返回出来的对象就是r，然后调用了r的run方法，然后把ZygoteInit的main方法返回掉，于是接下来就是`SystemServer.main`。

### SystemServer.main

```java
new SystemServer().run();
```

就调用了这么个方法

### SystemServer.run

```java
private void run() {
    try {
        ...
        String timezoneProperty = SystemProperties.get("persist.sys.timezone");
        if (timezoneProperty == null || timezoneProperty.isEmpty()) {
            Slog.w(TAG, "Timezone not set; setting to GMT.");
            SystemProperties.set("persist.sys.timezone", "GMT");
        }
        ...
        if (!SystemProperties.get("persist.sys.language").isEmpty()) {
            final String languageTag = Locale.getDefault().toLanguageTag();
            SystemProperties.set("persist.sys.locale", languageTag);
            SystemProperties.set("persist.sys.language", "");
            SystemProperties.set("persist.sys.country", "");
            SystemProperties.set("persist.sys.localevar", "");
        }
        ...
        Looper.prepareMainLooper();
        ...
        // Initialize the system context.
        createSystemContext();
        // Create the system service manager.
        mSystemServiceManager = new SystemServiceManager(mSystemContext
                                                         ...
        SystemServerInitThreadPool.get();
        );
    }
    finally {
        traceEnd(); // InitBeforeStartServices
    }
    ...
    // Start services.
    startBootstrapServices();
    startCoreServices();
    startOtherServices();
    ...
    // Loop forever.
    Looper.loop();
}
```

这里是真正运行System进程要初始化的工作的地方！！！

1. 先设置了一些时区、语言等等
1. 创建了一个消息循环Looper
1. 创建系统上下文、创建SystemServiceManager,创建一个线程池用来维护系统服务。
1. 调用各个方法启动各种系统服务，关于系统服务。
  + `startBootstrapServices`：这里启动一些依赖性比较强的服务。如安装器、设备标识服务、AMS、PackageManagerServices、UserManagerService、电池管理服务、Recovery服务、亮度服务、传感器等等等等。
  + `startCoreServices`：这里启动了UsageStatsService（用户使用情况服务）、WebViewUpdate服务
  + `startOtherServices`：一些杂七杂八的服务，比如闹钟、蓝牙、网络（包括wifi）、媒体、存储等等甚至是statusBar也是单开一个服务的，还有一个重要的是WindowymanagerServices，他在这里的原因是它要等到传感器都初始化好之后，它才能启动，这里顺便一提，还给AMS设置了一个回调systemReady，告诉AMS可以运行非系统的代码了。以及NotificationManager也是在这里创建的。
1. 启动Looper

**到这里System进程就启动完成啦。**