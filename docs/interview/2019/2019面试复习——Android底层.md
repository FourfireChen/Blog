# 2019面试复习——Android底层

## 四大组件

### Activity

#### 启动

从`startActivity`到新`Activity`的`onResume`算一个启动

> a1指源Activity
>
> a2指目标Activity

+ a1调用`Instrumentation`，通过`AIDL`，发送给`AMS`，参数包括：

  + `ApplicationThread`：本地Binder对象，供`AMS`回调a1进程
  + `Intent`：目的意图
  + a1的信息
  + token
  + 是否要返回启动结果

  > Instrumentation：用于监控应用和系统之间的交互

+ `AMS`解析`Intent`信息，得到a2部分信息，保存

+ `AMS`做情况分支

  + 是否原栈顶？

    + 是，是否需要重新启动？（启动模式相关）

  + 是否是同一个app下的Activity

  + 是否由权限启动：PMS

    **插件化要绕过的地方**

  将需要启动的Activity（不一定是a2了，可能是别的，这里假设a2）所在的`TaskRecord`放到Activity栈顶，`a2`放到`TaskRecord`顶

+ 调用`ActivityStack`对顶端Activity做处理

  > `resumeTopAcitvityInnerLocked`方法很臃肿，多次调用

+ IPC请求（通过发送过来的`ApplicationThread`进行）暂停a1，埋下一个超时炸弹。然后返回。（除非设置了flag：`FLAG_RESUME_WHILE_PAUSING`）

  > 此处的IPC应用在API28做了极大的架构调整，用了包装了一个策略模式。之前是AIDL。再之前是直接Binder调用
  >
  > 整个流程大概如下：
  >
  > + `AMS`通过`LifecycleManager`把`Item`（如PauseItem等）、`ApplicationThread`、`Token`包装成`Transaction`
  > + `Transaction`把自己通过`ApplicationThread`发送过去
  >
  > + `ApplicationThread`接收到请求后转发给`ActivityThread`，`ActivityThread`现在是继承了`ClientTransactionHandler`处理
  >
  > - `ClientTransactionHandler`调用`Transaction`对象的预处理`preExe`，然后通过`ActivityThread`的`mH`转发到专用的处理类`TransactionExecutor`
  >
  >   **此处做转发的意义是切换线程，从Binder线程切换到UI线程**
  >
  > - `TransactionExecutor`对`Transaction`的处理顺序是：
  >
  >   - `Transaction`的`Callback`：用于预处理
  >
  >     这个Callback也是一个Item，同样会调用到execute和postExe
  >
  >   - `Transaction`的`Item`的`execute`：用于正式处理
  >
  >     `XxxActivityItem`的`execute`，会调用`ActivityThread`的`handleXxxActivity`方法
  >
  >   - `Transaction`的`postExe`：用于回调通知`AMS`

  + 如上文所述，此处会调用到`handlePauseActivity`->`onPause`

    > 这里有一个有意思的现象就是onSaveInstanceState的调用时机

    > 关于生命周期的处理，会涉及到fragment的处理

  + `postExe`调用`AMS.activityPaused`

+ 拆除炸弹。然后继续启动a1

  + 进程已存在：同样通过`Transaction`，Launch作为Callback，Resume作为execute。按上文所述，则执行顺序为Launch->Resume
  + 进程不存在：创建进程
    + `ActivityThread`的main方法之后会通知`AMS`启动完成
    + `AMS`接收到消息后，重新通知`ActivityThread`做初始化，如`Application`、`Instrumentation`
    + `AMS`启动之前已放置在栈顶的a2，流程与进程存在时一样（这个流程和上一步是同步的）

+ `ActivityThread`接收到请求，照上文所述，依次执行

  + Callback：handleLaunchActivity

    + 创建`ContextImpl`、`Activity`实例也就是a2

      **`Instrumentation.newActivity`是插件化hook点之一**

    + 调用`Activity`的`attach`方法：设置`Context`、`Instrumentation`等等，创建`Window`对象

    + 调用`onCreate`

  + execute

    + cycleToPath：`handleStartActivity`->`onStart`->`onRestoreInstanceState`

      > Fragment的onStart似乎在Activity之后，但onCreate好像在之前

    + execute：`handleResumeActivity`

      + `onResume`
      + 设置`Activity`的`Window`、`DecorView`
      + 加入一个`Idler`空闲处理器：在空闲的时候发送消息给`AMS`，stop或destory之前的Activity

#### onSaveInstanceState/onRestoreInstanceState

+ onSaveInstanceState
  + 3.0之前：onPause之前
  + 9.0之前：onPause之前，onStop之后
  + 9.0之后：onStop之后
+ onRestoreInstanceState
  + onStart之后

#### ActivityStack

+ 栈

  + Task是栈的抽象，不指定的情况下，Activity以包名为栈名，同个app位于同个栈。但可以特殊指定`taskaffinity`
  + 整个AMS维护一个Stack，元素是Task，顶端是前台栈，其他是后台栈

+ 启动模式

  + standard：总会创建实例，位于启动者的栈中

    非Activity的Context启动会抛异常，因为没有栈。可以指定new_task位。但这里有个bug，Framework在api24-api27自动加了new_task位，导致也可以启动。

    **？**

  + singleTop：如果位于栈顶，不重复创建，且onNewIntent会调用。用于防止多次点击

  + singleTask：完全单例。如果已存在，会弹出上方其他Activity，并调用onNewIntent。会CLEAR_TOP。常用于主页、登录页

  + singleInstance：完全单例。单独位于一个栈



