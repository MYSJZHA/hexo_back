---
title: Activity启动模式，启动过程
date: 2018-07-26 14:20:07
tags: [Android,面试]
keywords: 启动模式,启动过程,Intent匹配规则,App启动流程
---
面试总结，关于Activity启动模式、启动过程，Intent匹配规则、App启动流程等
<!--more-->
##### 启动模式：
* standard：标准模式，这也是系统的默认模式。每次启动一个Activity都会重新创建一个新的实例，不管这个实例是否已经存在。
* singleTop：栈顶复用模式。在这种模式下，如果新Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时它的onNewIntent方法会被回调，通过此方法的参数我们可以取出当前请求的信息。需要注意的是，这个Activity的onCreate、onStart不会被系统调用，因为它并没有发生改变。如果新Activity的实例已存在但不是位于栈顶，那么新Activity仍然会重新重建。
* singleTask：栈内复用模式。这是一种单实例模式，在这种模式下，只要Activity在一个栈中存在，那么多次启动此Activity都不会重新创建实例，和singleTop一样，系统也会回调其onNewIntent。
* singleInstance：单实例模式。这是一种加强的singleTask模式，它除了具有singleTask模式的所有特性外，还加强了一点，那就是具有此种模式的Activity只能单独地位于一个任务栈中，

还有一个参数 `TaskAffinity`,这个参数标识了一个Activity所需要的任务栈的名字，默认情况下，所有Activity所需的任务栈的名字为应用的包名。当然，我们可以为每个Activity都单独指定TaskAffinity属性，这个属性值必须不能和包名相同，否则就相当于没有指定。TaskAffinity属性主要和singleTask启动模式或者allowTaskReparenting属性配对使用，在其他情况下没有意义。
还有Activity中能够影响启动模式、运行状态的标记位：

** FLAG_ACTIVITY_NEW_TASK **
这个标记位的作用是为Activity指定“singleTask”启动模式，其效果和在XML中指定该启动模式相同。
** FLAG_ACTIVITY_SINGLE_TOP **
这个标记位的作用是为Activity指定“singleTop”启动模式，其效果和在XML中指定该启动模式相同。
** FLAG_ACTIVITY_CLEAR_TOP **
具有此标记位的Activity，当它启动时，在同一个任务栈中所有位于它上面的Activity都要出栈。这个模式一般需要和FLAG_ACTIVITY_NEW_TASK配合使用，在这种情况下，被启动Activity的实例如果已经存在，那么系统会调用它的onNewIntent。如果被启动的Activity采用standard模式启动，那么它连同它之上的Activity都要出栈，系统会创建新的Activity实例并放入栈顶。通过1.2.1节中的分析可以知道，singleTask启动模式默认就具有此标记位的效果。
** FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS **
具有这个标记的Activity不会出现在历史Activity的列表中，当某些情况下我们不希望用户通过历史列表回到我们的Activity的时候这个标记比较有用。它等同于在XML中指定Activity的属性android:excludeFromRecents="true"。

##### Intent匹配规则
启动Activity分为两种，显式调用和隐式调用。显式调用需要明确地指定被启动对象的组件信息，包括包名和类名，而隐式调用则不需要明确指定组件信息。原则上一个Intent不应该既是显式调用又是隐式调用，如果二者
共存的话以显式调用为主。显式调用很简单，这里主要介绍一下隐式调用。隐式调用需要Intent能够匹配目标组件的IntentFilter中所设置的过滤信息，如果不匹配将无法启动目标Activity。
为了匹配过滤列表，需要同时匹配过滤列表中的action、category、data信息，否则匹配失败。一个过滤列表中的action、category和data可以有多个，所有的action、category、data分别构成不同类别，同一类别的信息共同约束当前类别的匹配过程。只有一个Intent同时匹配action类别、category类别、data类别才算完全匹配，只有完全匹配才能成功启动目标Activity。另外一点，一个Activity中可以有多个intent-filter，一个Intent只要能匹配任何一组intent-filter即可成功启动对应的Activity。

* action的匹配规则
action是一个字符串，系统预定义了一些action，同时我们也可以在应用中定义自己的action。action的匹配规则是Intent中的action必须能够和过滤规则中的action匹配，这里说的匹配是指action的字符串值完全一样。一个过滤规则中可以有多个action，那么只要Intent中的action能够和过滤规则中的任何一个action相同即可匹配成功。需要注意的是，Intent中如果没有指定action，那么匹配失败。另外，action区分大小写，大小写不同字符串相同的action会匹配失败。
* category的匹配规则
category是一个字符串，系统预定义了一些category，同时我们也可以在应用中定义自己的category。category的匹配规则和action不同，它要求Intent中如果含有category，那么所有的category都必须和过滤规则中的其中一个category相同。换句话说，Intent中如果出现了category，不管有几个category，对于每个category来说，它必须是过滤规则中已经定义了的category。当然，Intent中可以没有category，如果没有category的话，按照上面的描述，这个Intent仍然可以匹配成功。这里要注意下它和action匹配过程的不同，action是要求Intent中必须有一个action且必须能够和过滤规则中的某个action相同，而category要求
Intent可以没有category，但是如果你一旦有category，不管有几个，每个都要能够和过滤规则中的任何一个category相同。
* data的匹配规则
data的匹配规则和action类似，如果过滤规则中定义了data，那么Intent中必须也要定义可匹配的data。在介绍data的匹配规则之前，我们需要先了解一下data的结构，因为data稍微有些复杂
``` xml
<data   android:scheme="string"
        android:host="string"
        android:port="string"
        android:path="string"
        android:pathPattern="string"
        android:pathPrefix="string"
        android:mimeType="string" />
```
data由两部分组成，mimeType和URI。mimeType指媒体类型，比如image/jpeg、audio/mpeg4-generic和video/*等，可以表示图片、文本、视频等不同的媒体格式，而URI中包含的数据就比较多了，下面是URI的结构：
` <scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>] `
有如下过滤规则

` <data android:mimeType="image/*" /> `
这种规则指定了媒体类型为所有类型的图片，那么Intent中的mimeType属性必须为“image/*”才能匹配，这种情况下虽然过滤规则没有指定URI，但是却有默认值，URI的默认值为content和file。也就是说，虽然没有指定URI，但是Intent中的URI部分的schema必须为content或者file才能匹配，这点是需要尤其注意的。为了匹配上面中规则，我们可以写出如下示例
`intent.setDataAndType(Uri.parse("file://abc"),"image/png")。`
另外，如果要为Intent指定完整的data，必须要调用setDataAndType方法，不能先调用setData再调用setType，因为这两个方法彼此会清除对方的值。
最后，当我们通过隐式方式启动一个Activity的时候，可以做一下判断，看是否有Activity能够匹配我们的隐式Intent，如果不做判断就有可能出现上述的错误了。判断方法有两种：采用PackageManager的resolveActivity方法或者Intent的resolveActivity方法，如果它们找不到匹配的Activity就会返回null，我们通过判断返回值就可以规避上述错误了。另外，PackageManager还提供了queryIntentActivities方法，这个方法和resolveActivity方法不同的是：它不是返回最佳匹配的Activity信息而是返回所有成功匹配的Activity信息。
``` java
public abstract List<ResolveInfo> queryIntentActivities(Intent intent,int flags);
public abstract ResolveInfo resolveActivity(Intent intent,int flags);
```
上述两个方法的第一个参数比较好理解，第二个参数需要注意，我们要使用MATCH_DEFAULT_ONLY这个标记位，这个标记位的含义是仅仅匹配那些在intent-filter中声明了
`<category  android:name="android.intent.category.DEFAULT"/>`这个category的Activity。使用这个标记位的意义在于，只要上述两个方法不返回null，那么startActivity一定可以成功。如果不用这个标记位，就可以把intent-filter中category不含DEFAULT的那些Activity给匹配出来，从而导致startActivity可能失败。因为不含有DEFAULT这个category的Activity是无法接收隐式Intent的。

##### App启动过程

1. 点击桌面App图标，Launch进程采用Binder IPC向system_server进程发起startActivity请求
2. system_server收到请求后，向zygote进程发送创建进程请求。
3. Zygote进程fork出新的子进程，即App进程。
4. App进程通过Binder IPC向system_server进程发起attachApplication请求
5. system_server进程在收到请求后，进行一系列的准备工作，再通过Binder IPC向App进程发送scheduleLaunchActivity请求。
6. App进程的binder线程(ApplicationThread)在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；
7. 主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法；
8. 到此，App便正式启动，开始进入Activity生命周期。

** 涉及到的类 **

* `Activity` startActivity方法的真正实现在Activity中。
* `Instrumentation` 每一个应用程序只有一个Instrumentation对象，每个Activity内都有一个对该对象的引用。Instrumentation可以理解为应用进程的管家，ActivityThread要创建或暂停某个Activity时，都需要通过Instrumentation来进行具体的操作,用来辅助Activity完成启动Activity的过程。
* `ActivityThread`（包含ApplicationThread + ApplicationThreadNative + IApplicationThread）：真正启动Activity的实现都在这里,应用的入口类，系统通过调用main函数，开启消息循环队列。ActivityThread所在线程被称为应用的主线程（UI线程）。与ActivityManagerServices配合，一起完成Activity的管理工作。
* `ActivityManagerService` 简称AMS，服务端对象。AMS是Android中最核心的服务之一，主要负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块相类似，因此它在Android中非常重要，它本身也是一个Binder的实现类。
* `ActivityManagerProxy` AMS服务在当前进程的代理类，负责与AMS通信。
* `ApplicationThread` 用来实现ActivityManagerService与ActivityThread之间的交互。在ActivityManagerService需要管理相关Application中的Activity的生命周期时，通过ApplicationThread的代理对象与ActivityThread通讯。
* `ApplicationThreadProxy` 是ApplicationThread在服务器端的代理，负责和客户端的ApplicationThread通讯。AMS就是通过该代理与ActivityThread进行通信的。
* `ActivityStack` Activity在AMS的栈管理，用来记录已经启动的Activity的先后关系，状态信息等。通过ActivityStack决定是否需要启动新的进程。
* `ActivityRecord` ActivityStack的管理对象，每个Activity在AMS对应一个ActivityRecord，来记录Activity的状态以及其他的管理信息。其实就是服务器端的Activity对象的映像。
* `TaskRecord` AMS抽象出来的一个“任务”的概念，是记录ActivityRecord的栈，一个“Task”包含若干个ActivityRecord。AMS用TaskRecord确保Activity启动和退出的顺序。如果你清楚Activity的4种launchMode，那么对这个概念应该不陌生。

** 基本概念 **
###### zygote
Android是基于Linux系统的，而在Linux中，所有的进程都是由init进程直接或者是间接fork出来的，zygote进程也不例外。至于init进程怎么来的，可以搜一下Android系统启动过程。
在Android系统里面，zygote是一个进程的名字。Android是基于Linux System的，当你的手机开机的时候，Linux的内核加载完成之后就会启动一个叫“init“的进程。在Linux System里面，所有的进程都是由init进程fork出来的，我们的zygote进程也不例外。
我们都知道，每一个App其实都是
* 一个单独的虚拟机
* 一个单独的进程
所以当系统里面的第一个zygote进程运行之后，在这之后再开启App，就相当于开启一个新的进程。而为了实现资源共用和更快的启动速度，Android系统开启新进程的方式，是通过fork第一个zygote进程实现的。所以说，除了第一个zygote进程，其他应用所在的进程都是zygote的子进程，

###### SystemServer
它也是个进程，而且是由zygote进程fork出来的。系统里面重要的服务都是在这个进程里面开启的，比如`ActivityManagerService`、`PackageManagerService`、`WindowManagerService` 等等。在zygote开启的时候，会调用ZygoteInit.main()进行初始化：

``` java
/**
 * Startup class for the zygote process.
 *
 * Pre-initializes some classes, and then waits for commands on a UNIX domain
 * socket. Based on these commands, forks off child processes that inherit
 * the initial state of the VM.
 *
 * Please see {@link ZygoteConnection.Arguments} for documentation on the
 * client protocol.
 *
 */
```
从注释上也可以看出这个类主要是为了初始化某些参数。比如
``` java
static void preload(TimingsTraceLog bootTimingsTraceLog) {
        Log.d(TAG, "begin preload");
        bootTimingsTraceLog.traceBegin("BeginIcuCachePinning");
        beginIcuCachePinning();
        bootTimingsTraceLog.traceEnd(); // BeginIcuCachePinning
        bootTimingsTraceLog.traceBegin("PreloadClasses");
        preloadClasses();
        bootTimingsTraceLog.traceEnd(); // PreloadClasses
        bootTimingsTraceLog.traceBegin("PreloadResources");
        preloadResources();
        bootTimingsTraceLog.traceEnd(); // PreloadResources
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadAppProcessHALs");
        nativePreloadAppProcessHALs();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadOpenGL");
        preloadOpenGL();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        preloadSharedLibraries();
        preloadTextResources();
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        WebViewFactory.prepareWebViewInZygote();
        endIcuCachePinning();
        warmUpJcaProviders();
        Log.d(TAG, "end preload");

        sPreloadComplete = true;
    }

    public static void lazyPreload() {
        Preconditions.checkState(!sPreloadComplete);
        Log.i(TAG, "Lazily preloading resources.");

        preload(new TimingsTraceLog("ZygoteInitTiming_lazy", Trace.TRACE_TAG_DALVIK));
    }
```
还有一些关键的方法`preloadSharedLibraries()`、`preloadOpenGL()`、`preloadTextResources()`、`preloadClasses()`、`preloadResources()`、`preloadDrawables()`、`preloadColorStateLists()` 等.还有一个`startSystemServer()`方法。

###### ActivityManagerService
简称AMS,服务端对象，负责系统中所有Activity生命周期。它的初始化时机很明确，就是在SystemServer进程开启的时候，就会初始化ActivityManagerService。具体情况可以看一下`SystemServer.java`类。
经过上面这些步骤，我们的ActivityManagerService对象已经创建好了，并且完成了成员变量初始化。而且在这之前，调用createSystemContext()创建系统上下文的时候，也已经完成了mSystemContext和ActivityThread的创建。注意，这是系统进程开启时的流程，在这之后，会开启系统的Launcher程序，完成系统界面的加载与显示。

###### 为什么说AMS是服务端对象

其实服务器客户端的概念不仅仅存在于Web开发中，在Android的框架设计中，使用的也是这一种模式。服务器端指的就是所有App共用的系统服务，比如我们这里提到的ActivityManagerService，和前面提到的PackageManagerService、WindowManagerService等等，这些基础的系统服务是被所有的App公用的，当某个App想实现某个操作的时候，要告诉这些系统服务，比如你想打开一个App，那么我们知道了包名和MainActivity类名之后就可以打开

``` java
Intent intent = new Intent(Intent.ACTION_MAIN);  
intent.addCategory(Intent.CATEGORY_LAUNCHER);
ComponentName cn = new ComponentName(packageName, className);
intent.setComponent(cn);  
startActivity(intent);
```

但是，我们的App通过调用startActivity()并不能直接打开另外一个App，这个方法会通过一系列的调用，最后还是告诉AMS说：“我要打开这个App，我知道他的住址和名字，你帮我打开吧！”所以是AMS来通知zygote进程来fork一个新进程，来开启我们的目标App的。这就像是浏览器想要打开一个超链接一样，浏览器把网页地址发送给服务器，然后还是服务器把需要的资源文件发送给客户端的。

知道了Android Framework的客户端服务器架构之后，我们还需要了解一件事情，那就是我们的App和AMS(SystemServer进程)还有zygote进程分属于三个独立的进程，他们之间如何通信呢？
App与AMS通过Binder进行IPC通信，AMS(SystemServer进程)与zygote通过Socket进行IPC通信。
那么AMS有什么用呢？在前面我们知道了，如果想打开一个App的话，需要AMS去通知zygote进程，除此之外，其实所有的Activity的开启、暂停、关闭都需要AMS来控制，所以我们说，AMS负责系统中所有Activity的生命周期。
在Android系统中，任何一个Activity的启动都是由AMS和应用程序进程（主要是ActivityThread）相互配合来完成的。AMS服务统一调度系统中所有进程的Activity启动，而每个Activity的启动过程则由其所属的进程具体来完成。

###### Launcher

当我们点击手机桌面上的图标的时候，App就由Launcher开始启动了。Launcher本质上也是一个应用程序，和我们的App一样，也是继承自Activity，系统源码可以在这里看 http://androidxref.com/8.0.0_r4/xref/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
``` java
View createShortcut(int layoutResId, ViewGroup parent, ShortcutInfo info) {
        BubbleTextView favorite = (BubbleTextView) mInflater.inflate(layoutResId, parent, false);
        favorite.applyFromShortcutInfo(info, mIconCache);
        favorite.setOnClickListener(this);
        return favorite;
}
```
创建图标并设置点击监听
``` java
/**
     *
     * Launches the intent referred by the clicked shortcut.
     * @param v The view representing the clicked shortcut.
     */
    public void onClick(View v) {
        // Make sure that rogue clicks don't get through while allapps is launching, or after the
        // view has detached (it's possible for this to happen if the view is removed mid touch).
        if (v.getWindowToken() == null) {
            return;
        }

        if (!mWorkspace.isFinishedSwitchingState()) {
            return;
        }

        Object tag = v.getTag();
        if (tag instanceof ShortcutInfo) {
            // Open shortcut
            final Intent intent = ((ShortcutInfo) tag).intent;
            int[] pos = new int[2];
            v.getLocationOnScreen(pos);
            intent.setSourceBounds(new Rect(pos[0], pos[1],
                    pos[0] + v.getWidth(), pos[1] + v.getHeight()));

            boolean success = startActivitySafely(v, intent, tag);

            if (success && v instanceof BubbleTextView) {
                mWaitingForResume = (BubbleTextView) v;
                mWaitingForResume.setStayPressed(true);
            }
        } else if (tag instanceof FolderInfo) {
            if (v instanceof FolderIcon) {
                FolderIcon fi = (FolderIcon) v;
                handleFolderClick(fi);
            }
        } else if (v == mAllAppsButton) {
            if (isAllAppsVisible()) {
                showWorkspace(true);
            } else {
                onClickAllAppsButton(v);
            }
        }
    }

```
从上面可以看到，在桌面上点击快捷图标的时候，会调用
``` java
startActivitySafely(v, intent, tag);
```
具体代码就不抄了，看一下上面的链接中的源码就好，在该方法中调用了`startActivity(v, intent, tag)`，这里会调用Activity.startActivity(intent, opts.toBundle())，这个方法熟悉吗？这就是我们经常用到的Activity.startActivity(Intent)的重载函数。而且由于设置了
``` java
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
```
所以这个Activity会添加到一个新的Task栈中，而且，startActivity()调用的其实是startActivityForResult()这个方法。
所以现在明确了，Launcher中开启一个App，其实和我们在Activity中直接startActivity()基本一样，都是调用了Activity.startActivityForResult()。

###### Instrumentation
每个Activity都持有Instrumentation对象的一个引用，但是整个进程只会存在一个Instrumentation对象。当startActivityForResult()调用之后，实际上还是调用了mInstrumentation.execStartActivity().
下面是mInstrumetation.execStartActivity()的实现
``` java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }

```
这里的 ActivityManagerNative.getDefault 返回ActivityManagerService的远程接口，即 ActivityManagerProxy 接口，有人可能会问了为什么会是ActivityManagerProxy，这就涉及到Binder通信了，这里不再展开。通过Binder驱动程序， ActivityManagerProxy 与AMS服务通信，则实现了跨进程到System进程。

``` java
 /**
     * Retrieve the system's default/global activity manager.
     */
    static public IActivityManager getDefault() {
        return gDefault.get();
    }

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };

    /**
     * Cast a Binder object into an activity manager interface, generating
     * a proxy if needed.
     */
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }
```
###### AMS响应Launcher进程请求
至此，点击桌面图标调用startActivity()，终于把数据和要开启Activity的请求发送到了AMS了,AMS收到startActivity的请求之后，会按照如下的方法链进行调用：
``` java
@Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }

    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
```
这里又出现了一个`mStackSupervisor`，定义是这么说的
``` java
/** Run all ActivityStacks through this */
    ActivityStackSupervisor mStackSupervisor;

```
在`mStackSupervisor.startActivityMayWait()`方法中又调用了`startActivityLocked()`方法，接着调用了`startActivityUncheckedLocked()`方法，在这个方法中一大堆眼花缭乱的判断，最终调用了`targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options)`方法，然后调用了`mStackSupervisor.resumeTopActivitiesLocked(this, r, options)`方法，然后调用`result = targetStack.resumeTopActivityLocked(target, targetOptions)`方法，调用`result = resumeTopActivityInnerLocked(prev, options)`方法，在这个方法里，prev.app为记录启动Lancher进程的ProcessRecord，prev.app.thread为Lancher进程的远程调用接口IApplicationThead，所以可以调用prev.app.thread.schedulePauseActivity，到Lancher进程暂停指定Activity。至此，AMS对Launcher的请求已经响应，这是我们发现又通过Binder通信回调至Launcher进程

######  Launcher进程挂起Launcher，再次通知AMS

看一下怎么挂起Launcher的,在ActivityThread中：
``` java
private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
            if (userLeaving) {
                performUserLeavingActivity(r);
            }

            r.activity.mConfigChangeFlags |= configChanges;
            performPauseActivity(token, finished, r.isPreHoneycomb());

            // Make sure any pending writes are now committed.
            if (r.isPreHoneycomb()) {
                QueuedWork.waitToFinish();
            }

            // Tell the activity manager we have paused.
            if (!dontReport) {
                try {
                    ActivityManagerNative.getDefault().activityPaused(token);
                } catch (RemoteException ex) {
                }
            }
            mSomeActivitiesChanged = true;
        }
    }
```
这部分Launcher的ActivityThread处理页面Paused并且再次通过ActivityManagerProxy通知AMS。

###### AMS创建新的进程

创建新进程的时候，AMS会保存一个ProcessRecord信息，如果应用程序中的AndroidManifest.xml配置文件中，我们没有指定Application标签的process属性，系统就会默认使用package的名称。每一个应用程序都有自己的uid，因此，这里uid + process的组合就可以为每一个应用程序创建一个ProcessRecord。
在`ActivityManagerService`中，
``` java
private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        long startTime = SystemClock.elapsedRealtime();
        ......
        // Start the process.  It will either succeed and return a result containing
            // the PID of the new process, or else throw a RuntimeException.
            boolean isActivityProcess = (entryPoint == null);
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);
            checkTime(startTime, "startProcess: asking zygote to start proc");
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            checkTime(startTime, "startProcess: returned from zygote!");
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

            if (app.isolated) {
                mBatteryStatsService.addIsolatedUid(app.uid, app.info.uid);
            }
            mBatteryStatsService.noteProcessStart(app.processName, app.info.uid);
            checkTime(startTime, "startProcess: done updating battery stats");
```
这里主要是调用Process.start接口来创建一个新的进程，新的进程会导入android.app.ActivityThread类，并且执行它的main函数，这就是每一个应用程序都有一个ActivityThread实例来对应的原因。

###### 应用进程初始化
来看Activity的main函数，这里绑定了主线程的Looper，并进入消息循环，大家应该知道，整个Android系统是消息驱动的，这也是为什么主线程默认绑定Looper的原因：
``` java
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        AndroidKeyStoreProvider.install();

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
attach函数最终调用了ActivityManagerService的远程接口ActivityManagerProxy的attachApplication函数，传入的参数是mAppThread，这是一个ApplicationThread类型的Binder对象，它的作用是AMS与应用进程进行进程间通信的。
将进程和指定的Application绑定起来。这个是通过上节的ActivityThread对象中调用bindApplication()方法完成的。该方法发送一个BIND_APPLICATION的消息到消息队列中, 最终通过handleBindApplication()方法处理该消息. 然后调用makeApplication()方法来加载App的classes到内存中。

###### 在AMS中注册应用进程，启动栈顶页面

mMainStack.topRunningActivityLocked(null)从堆栈顶端取出要启动的Activity，并在realStartActivityLockedhan函数中通过ApplicationThreadProxy调回App进程启动页面。
在`ActivityStackSupervisor`中
``` java
final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {
                    app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
            }
```
它会调用application线程对象中的scheduleLaunchActivity()发送一个LAUNCH_ACTIVITY消息到消息队列中, 通过 handleLaunchActivity()来处理该消息。在 handleLaunchActivity()通过performLaunchActiivty()方法回调Activity的onCreate()方法和onStart()方法，然后通过handleResumeActivity()方法，回调Activity的onResume()方法，而后会通知AMS该MainActivity已经处于resume状态最终显示Activity界面。
至此，整个启动流程告一段落。


最后：
###### 一个App的程序入口到底是什么？
是ActivityThread.main()。

###### 整个App的主线程的消息循环是在哪里创建的？
是在ActivityThread初始化的时候，就已经创建消息循环了，所以在主线程里面创建Handler不需要指定Looper，而如果在其他线程使用Handler，则需要单独使用Looper.prepare()和Looper.loop()创建消息循环。可以看ActivityThread的main方法

###### Application是在什么时候创建的？onCreate()什么时候调用的？
也是在ActivityThread.main()的时候，就是在thread.attach(false)的时候。

``` java
if (!system) {
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
}
```
这里需要关注的就是mgr.attachApplication(mAppThread)，这个就会通过Binder调用到AMS里面对应的方法:
``` java
    @Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```
然后调用的就是`private final boolean attachApplicationLocked(IApplicationThread thread,int pid)`方法，thread是IApplicationThread，实际上就是ApplicationThread在服务端的代理类ApplicationThreadProxy，然后又通过IPC就会调用到ApplicationThread的对应方法。这个方法里面又调用了`sendMessage()`，里面有函数的编号H.BIND_APPLICATION，然后这个Messge会被H这个Handler处理:
``` java
private class H extends Handler {

    public static final int BIND_APPLICATION        = 110;
    case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
}
```
然后在`handleBindApplication(data)`方法中
``` java
 try {
      java.lang.ClassLoader cl = instrContext.getClassLoader();
      mInstrumentation = (Instrumentation)cl.loadClass(data.instrumentationName.getClassName()).newInstance();
} catch (Exception e) {
        throw new RuntimeException("Unable to instantiate instrumentation "+ data.instrumentationName + ": " + e.toString(), e);

        ......
         // Do this after providers, since instrumentation tests generally start their
            // test thread at this point, and we don't want that racing.
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
}
```
不同的版本代码不尽相同，但是基本逻辑不会变。
参考、抄袭的链接如下：
https://blog.csdn.net/bfboys/article/details/52564531
https://www.jianshu.com/p/a72c5ccbd150
https://www.jianshu.com/p/6037f6fda285
https://www.jianshu.com/p/a72c5ccbd150

----
以上


