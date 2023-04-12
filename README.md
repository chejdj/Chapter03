# Chapter03

**这个 Sample 难度的确有点大，我们可以看着例子中的一些 Hook 点去找源码中对应的实现，看看为什么不同的版本要使用不同的实现方式。**

该例子主要实现了在运行时动态获取对象分配的情况，可以运用在自动化的分析中。

目的是展示一种模仿 Android Profiler，但可以脱离 Profiler 实现自动化内存分配分析的方案。

我们可以在它基础上做很多自定义的操作：
1. 追踪Bitmap的创建堆栈；
2. 追踪线程的创建堆栈；
3. 追踪特定对象的分配；
4. ...

在后面我们也会有大量的这些例子，只要充分清楚底层原理，我们可以做到事情还是很多的。

**在高版本 Android Profiler 使用了新的方法来实现 Allocation Tracker，这块我们后面在的章节会给出新的实现。**

开发环境
=======
Android Studio 3.2.1
NDK 16~19

运行环境
======
支持 ARMv7a 和 x86平台
项目支持Dalvik 和 Art 虚拟机，理论上兼容了4.0到9.0的机型，在7.1到9.0的手机和模拟器上已经跑通，比较稳定。
由于国内机型的差异化，无法适配所有机型，项目只做展示原理使用，并不稳定。

使用方式
======

1. 点击`开始记录`即可触发对象分配记录会在 logcat 中看见如下日志
```
/com.dodola.alloctrack I/AllocTracker: ====current alloc count 111=====
```
说明已经开始记录对象的分配

2. 当对象达到设置的最大数量的时候触发内存 dump,会有如下日志
```
com.dodola.alloctrack I/AllocTracker: saveARTAllocationData /data/user/0/com.dodola.alloctrack/files/1544005106 file, fd: 63
com.dodola.alloctrack I/AllocTracker: saveARTAllocationData write file to /data/user/0/com.dodola.alloctrack/files/1544005106
```


3. 数据会保存在 `/data/data/com.dodola.alloctrack/files`目录下

4. 数据解析。采集下来的数据无法直接通过编辑器打开，需要通过 dumpprinter 工具来进行解析，操作如下
```
dump 工具存放在tools/DumpPrinter-1.0.jar 中

可以通过 gradle task :buildAlloctracker.jar编译

//调用方法：
java -jar tools/DumpPrinter-1.0.jar dump文件路径 > dump_log.txt
```

5. 然后就可以在 `dump_log.txt` 中看到解析出来的数据
采集到的数据基本格式如下：

```
Found 10240 records://dump 下来的数据包含对象数量
tid=7205 java.lang.Class (4144 bytes)//当前线程  类名  分配的大小
		//下面是分配该对象的时候当前线程的堆栈信息
    android.support.v7.widget.Toolbar.ensureMenuView (Toolbar.java:1047)
    android.support.v7.widget.Toolbar.setMenu (Toolbar.java:551)
    android.support.v7.widget.ToolbarWidgetWrapper.setMenu (ToolbarWidgetWrapper.java:370)
    android.support.v7.widget.ActionBarOverlayLayout.setMenu (ActionBarOverlayLayout.java:721)
    android.support.v7.app.AppCompatDelegateImpl.preparePanel (AppCompatDelegateImpl.java:1583)
    android.support.v7.app.AppCompatDelegateImpl.doInvalidatePanelMenu (AppCompatDelegateImpl.java:1869)
    android.support.v7.app.AppCompatDelegateImpl$2.run (AppCompatDelegateImpl.java:230)
    android.os.Handler.handleCallback (Handler.java:792)
    android.os.Handler.dispatchMessage (Handler.java:98)
    android.os.Looper.loop (Looper.java:176)
    android.app.ActivityThread.main (ActivityThread.java:6701)
    java.lang.reflect.Method.invoke (Native method)
    com.android.internal.os.Zygote$MethodAndArgsCaller.run (Zygote.java:246)
    com.android.internal.os.ZygoteInit.main (ZygoteInit.java:783)
```

原理解析
======
项目使用了 inline hook 来拦截内存对象分配时候的 `RecordAllocation` 函数，通过拦截该接口可以快速获取到当时分配对象的类名和分配的内存大小。

在初始化的时候我们设置了一个分配对象数量的最大值，如果从 start 开始对象分配数量超过最大值就会触发内存 dump，然后清空 alloc 对象列表，重新计算。该功能和  Android Studio 里的 Allocation Tracker 类似，只不过可以在代码级别更细粒度的进行控制。可以精确到方法级别。

核心点如下：

1. 阅读源码。各个ROM版本的实现有所差异，我们需要在各个版本上面找到合适的 Hook 点。这里需要我们对整个流程和原理都比较清楚。
2. 选择合适的框架。需要对各种Hook框架有清楚的认识，如何选择，如何使用。

这套方案的确有点复杂，Android Profiler 换了新的实现方案。整体实现会简单很多，后续也会给出实现。

Thanks
======
Substrate 一款经典的 hook 框架

[fbjni](https://github.com/facebookincubator/profilo/tree/master/deps/fbjni) 是从 Facebook 开源的一款jni工具类库，主要提供了工具类，ref utils ，Global JniEnv。

[ndk_dlopen](https://github.com/rrrfff/ndk_dlopen) 提供了force dlopen 机制

前提知识：
1. Java可以用反射来hook，C++没有反射但是同样可以hook。MSHookFunction 就是这样一个框架，支持hook C/C++代码， http://www.cydiasubstrate.com/api/c/MSHookFunction/
2. ndk_dlopen 和 ndk_dlsym 。前者是用来获取动态链接库，后者是通过获取到前者的动态链接库之后获取函数地址的。
3. C/C++在链接之前会把函数的名字mangled, 我们在ndk_dlsym函数里面需要填mangled之后的函数名， -》 _ZN3art3Dbg21DumpRecentAllocationsEnv
4. facebook::jni::这个应该是调用文章中说到的那个facebook的jni的开源框架。 JNIEnv *env 这个是一个全局的JNI环境变量，储存了变量，还有很多JNI的函数供开发者使用。比如代码中的env->GetArrayLength(saveData.data);

调用流程就是：
1. 首先先调用了tracker.initForArt， 并且调用了JNI方法initForArt。所谓的初始化其实就是使用ndk_dlsym 拿到各个要hook的函数。比如artSetAllocTrackingEnable ，就是开启/关闭tracking的，源代码中有一行:

artSetAllocTrackingEnable = (void (*)(bool)) ndk_dlsym(libHandle,
"_ZN3art3Dbg23SetAllocTrackingEnabledEb");

这里注意并没有开启函数，只是拿到函数句柄而已。
2. void hookFunc() , 这个方法就是真正的把系统的tracking 函数hook调的地方。

比如 先调用 void *hookRecordAllocation26 = ndk_dlsym(handle,
"_ZN3art2gc20AllocRecordObjectMap16RecordAllocationEPNS_6ThreadEPNS_6ObjPtrINS_6mirror6ObjectEEEj");

这里注意是拿到函数的地址而不是句柄。

然后 调用hook -> MSHookFunction(hookRecordAllocation26, (void *) &newArtRecordAllocation26,
(void **) &oldArtRecordAllocation26);
通过这个函数把newArtRecordAllocation26 hook进原函数地址里，同时拿到旧函数的实现并导向oldArtRecordAllocation26。保留旧函数的原因是还需要使用旧函数的一些功能。

3. 最后我们可以看到hook的新函数，做了一个大小的判断。 if (allocObjectCount > setAllocRecordMax)， 如果大于setAllocRecordMax，就
jbyteArray allocData = getARTAllocatio nData();
SaveAllocationData saveData{allocData};
saveARTAllocationData(saveData);

以上代码就是把数据保存在log文件里面的实现。
