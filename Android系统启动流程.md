### Android系统启动流程

系统中及其重要的流程：

- init进程：Linux系统中用户空间的第一个进程，Init.main
- zygote进程：Java进程的鼻祖，所有App进程的父进程，ZygoteInit.main
- system_server进程：系统服务的载体，SystemServer.main
- servicemanager进程：binder服务的大管家，守护进程循环运行在binder_loop
- app进程：通过Process.start启动App进程，ActivityThread.main

### 一、init进程启动

init进程是Linux用户空间的第一个进程，进程号pid=1。Linux内核启动后会调用init.cpp的main方法。

主要作用：

- 创建和挂载启动所需的文件目录
- 解析init.rc配置文件并启动zygote进程、servicemanager进程
- 初始化和启动属性服务
- 处理子进程的终止

### 二、servicemanager进程启动

ServiceManager是由[init进程](http://gityuan.com/2016/02/05/android-init/)通过解析init.rc文件而创建的，启动Service Manager的入口函数是service_manager.c中的main()方法。

ServiceManager是Binder IPC通信过程中的守护进程，本身也是一个Binder服务，但并没有采用libbinder中的多线程模型来与Binder驱动通信，而是自行编写了binder.c直接和Binder驱动来通信，并且只有一个循环binder_loop来进行读取和处理事务，这样的好处是简单而高效。

**ServiceManager启动流程：**

1. 打开binder驱动，并调用mmap()方法分配128k的内存映射空间：binder_open();
2. 通知binder驱动使其成为守护进程：binder_become_context_manager()；
3. 进入循环状态，等待Client端的请求：binder_loop()。
4. 注册服务的过程，根据服务名称，但同一个服务已注册，重新注册前会先移除之前的注册信息；
5. 死亡通知: 当binder所在进程死亡后,会调用binder_release方法,然后调用binder_node_release.这个过程便会发出死亡通知的回调.

ServiceManager最核心的两个功能为查询和注册服务：

- 注册服务：记录服务名和handle信息，保存到svclist列表；
- 查询服务：根据服务名查询相应的的handle信息。

### 三、Zygote进程启动

Zygote是由init进程通过解析init.zygote.rc文件而创建的，启动入口App_main.cpp的main方法

Zygote启动过程函数调用流程：

``` 
App_main.main() -> AndroidRuntime.start() -> startVm() -> startReg() 

->通过JNI调用进入Java层👇

->ZygoteInit.main -> registerZygoteSocket() -> preload() -> startSystemServer() -> runSelectLoop()
```

#### 3.1、App_main.main()

启动参数解析、设置进程名、启动AppRuntime。

#### 3.2、start() / [-> AndroidRuntime.cpp]

- 调用startVm方法创建并启动Java虚拟机
- 调用startReg为Java虚拟机注册JNI方法
- 通过JNI调用ZygoteInit.main方法，进入Java层

#### 3.3、startVm()

创建Java虚拟机，并设置虚拟机参数和调优

#### 3.4、startReg()

注册JNI方法，比如：

```c++
    { "nativeZygoteInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeZygoteInit },
```

注册完成后，start()方法会通过JNI调用ZygoteInit.main()方法，进入Java层。

#### 进入Java层

#### 3.5、ZygoteInit.main()

```java
public static void main(String argv[]) {
    try {
        RuntimeInit.enableDdms(); //开启DDMS功能
        boolean startSystemServer = false;
        String socketName = "zygote";
        ...
        registerZygoteSocket(socketName); //为Zygote注册socket
        preload(); // 预加载类和资源
        SamplingProfilerIntegration.writeZygoteSnapshot();
        gcAndFinalize(); //GC操作
        if (startSystemServer) {
            startSystemServer(abiList, socketName);//启动system_server
        }
        runSelectLoop(abiList); //进入循环模式
        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run(); //启动system_server中会讲到。
    } catch (RuntimeException ex) {
        closeServerSocket();
        throw ex;
    }
}
```

为Zygote注册一个Socket服务端，地址为zygote，并进入循环模式

#### 3.6、registerZygoteSocket()

为Zygote进程创建一个Socket的本地服务端

#### 3.7、preload()

预加载资源

```java
static void preload() {
    //预加载位于/system/etc/preloaded-classes文件中的类
    preloadClasses();
    //预加载资源，包含drawable和color资源
    preloadResources();
    //预加载OpenGL
    preloadOpenGL();
    //通过System.loadLibrary()方法，
    //预加载"android","compiler_rt","jnigraphics"这3个共享库
    preloadSharedLibraries();
    //预加载 文本连接符资源
    preloadTextResources();
    //仅用于zygote进程，用于内存共享的进程
    WebViewFactory.prepareWebViewInZygote();
}
```

#### 3.9、startSystemServer()

启动systemServer进程

```java
private static boolean startSystemServer(String abiList, String socketName)
        throws MethodAndArgsCaller, RuntimeException {
    ...

    // fork子进程system_server
    pid = Zygote.forkSystemServer(
            parsedArgs.uid, parsedArgs.gid,
            parsedArgs.gids,
            parsedArgs.debugFlags,
            null,
            parsedArgs.permittedCapabilities,
            parsedArgs.effectiveCapabilities);
    ...

    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        //进入system_server进程
        handleSystemServerProcess(parsedArgs);
    }
    return true;
}
```

#### 3.10、runSelectLoop()

```java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
    //sServerSocket是socket通信中的服务端，即zygote进程。保存到fds[0]
    fds.add(sServerSocket.getFileDescriptor());
    peers.add(null);

    while (true) {
        StructPollfd[] pollFds = new StructPollfd[fds.size()];
        for (int i = 0; i < pollFds.length; ++i) {
            pollFds[i] = new StructPollfd();
            pollFds[i].fd = fds.get(i);
            pollFds[i].events = (short) POLLIN;
        }
        try {
             //处理轮询状态，当pollFds有事件到来则往下执行，否则阻塞在这里
            Os.poll(pollFds, -1);
        } catch (ErrnoException ex) {
            ...
        }
        
        for (int i = pollFds.length - 1; i >= 0; --i) {
            //采用I/O多路复用机制，当接收到客户端发出连接请求 或者数据处理请求到来，则往下执行；
            // 否则进入continue，跳出本次循环。
            if ((pollFds[i].revents & POLLIN) == 0) {
                continue;
            }
            if (i == 0) {
                //即fds[0]，代表的是sServerSocket，则意味着有客户端连接请求；
                // 则创建ZygoteConnection对象,并添加到fds。
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor()); //添加到fds.
            } else {
                //i>0，则代表通过socket接收来自对端的数据，并执行相应操作【见小节3.6】
                boolean done = peers.get(i).runOnce();
                if (done) {
                    peers.remove(i);
                    fds.remove(i); //处理完则从fds中移除该文件描述符
                }
            }
        }
    }
}
```

Zygote采用高效的I/O多路复用机制，保证在没有客户端连接请求或数据处理时休眠，否则响应客户端的请求。

#### 9.11、runOnece()

```java
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

    String args[];
    Arguments parsedArgs = null;
    FileDescriptor[] descriptors;

    try {
        //读取socket客户端发送过来的参数列表
        args = readArgumentList();
        descriptors = mSocket.getAncillaryFileDescriptors();
    } catch (IOException ex) {
        ...
        return true;
    }
    ...

    try {
        //将binder客户端传递过来的参数，解析成Arguments对象格式
        parsedArgs = new Arguments(args);
        ...
        // fork自身创建子进程
        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                parsedArgs.appDataDir);
    } catch (Exception e) {
        ...
    }

    try {
        if (pid == 0) {
            //子进程执行
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;
            //进入子进程流程
            handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
            return true;
        } else {
            //父进程执行
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
        }
    } finally {
        IoUtils.closeQuietly(childPipeFd);
        IoUtils.closeQuietly(serverPipeFd);
    }
}
```

创建进程成功后即pid==0，会通过反射执行ActivityThrea的main方法，执行进程创建

#### zygote进程启动总结

1. 解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法；
2. 调用AndroidRuntime的startVM()方法创建虚拟机，再调用startReg()注册JNI函数；
3. 通过JNI方式调用ZygoteInit.main()，第一次进入Java世界；
4. registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求；
5. preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView，用于提高app启动效率；
6. 通过startSystemServer()，fork system_server进程，也是上层framework的运行载体。
7. zygote功成身退，调用runSelectLoop()，随时待命，当接收到请求创建新进程请求时立即唤醒并执行相应工作。

### 四、SystemServer进程启动

#### 4.1、startSystemServer()

```java
        ...
        //准备参数
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
			...
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;
        int pid;
        try {
            //解析参数，生成目标格式
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            /* Request to fork the system server process */
            //fork 子进程 system_server
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        //进入子进程 system_server
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }
            //关闭Zygote进程创建的socket
            zygoteServer.closeServerSocket();
            //完成system_server进程剩余部分
            handleSystemServerProcess(parsedArgs);
        }
        return true;
    }

```

- Zygote进程通过fork自身得到子进程SystemServer进程
- 调用`handleSystemServerProcess`方法启动SystemServer进程

#### 4.2、handleSystemServerProcess

启动system_server进程

```java
    /**     * Finish remaining work for the newly forked system server process.     */    private static void handleSystemServerProcess(            ZygoteConnection.Arguments parsedArgs)            throws Zygote.MethodAndArgsCaller {        // set umask to 0077 so new files and directories will default to owner-only permissions.        Os.umask(S_IRWXG | S_IRWXO);        if (parsedArgs.niceName != null) {            Process.setArgV0(parsedArgs.niceName);//设置进程名 system_server        }        final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");        if (systemServerClasspath != null) {            performSystemServerDexOpt(systemServerClasspath);            // Capturing profiles is only supported for debug or eng builds since selinux normally            // prevents it.            boolean profileSystemServer = SystemProperties.getBoolean(                    "dalvik.vm.profilesystemserver", false);            if (profileSystemServer && (Build.IS_USERDEBUG || Build.IS_ENG)) {                try {                    File profileDir = Environment.getDataProfilesDePackageDirectory(                            Process.SYSTEM_UID, "system_server");                    File profile = new File(profileDir, "primary.prof");                    profile.getParentFile().mkdirs();                    profile.createNewFile();                    String[] codePaths = systemServerClasspath.split(":");                    VMRuntime.registerAppInfo(profile.getPath(), codePaths);                } catch (Exception e) {                    Log.wtf(TAG, "Failed to set up system server profile", e);                }            }        }        if (parsedArgs.invokeWith != null) {            ...            //启动进程            WrapperInit.execApplication(parsedArgs.invokeWith,                    parsedArgs.niceName, parsedArgs.targetSdkVersion,                    VMRuntime.getCurrentInstructionSet(), null, args);        } else {            ClassLoader cl = null;            if (systemServerClasspath != null) {                //创建类加载器，并赋予当前线程                cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);                Thread.currentThread().setContextClassLoader(cl);            }            /*             * Pass the remaining arguments to SystemServer.             */             //将其余参数床底给system_server            ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);        }    }
```

#### 4.3、zygoteInit()

进入`ZygoteInit.zygoteInit()`方法

```java
    public static final void zygoteInit(int targetSdkVersion, String[] argv,            ClassLoader classLoader) throws Zygote.MethodAndArgsCaller {        if (RuntimeInit.DEBUG) {            Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");        }        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");        RuntimeInit.redirectLogStreams();        RuntimeInit.commonInit();        //启动Binder线程池        ZygoteInit.nativeZygoteInit();        //进入SystemServer的main方法        RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);    }
```

- 启动Binder线程池，Native方法
- 进入SystemServer的main方法。嗲用RuntimeInit.applicationInit

#### 4.4、RuntimeInit.applicationInit()

```java
    private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)            throws ZygoteInit.MethodAndArgsCaller {            nativeSetExitWithoutCleanup(true);        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);        final Arguments args;        try {            args = new Arguments(argv);        } catch (IllegalArgumentException ex) {            Slog.e(TAG, ex.getMessage());            // let the process exit            return;        }        // The end of of the RuntimeInit event (see #zygoteInit).        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);        // Remaining arguments are passed to the start class's static main        invokeStaticMain(args.startClass, args.startArgs, classLoader);    }    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)            throws ZygoteInit.MethodAndArgsCaller {        Class<?> cl;        try {            //通过反射得到SystemServer类            cl = Class.forName(className, true, classLoader);        } catch (ClassNotFoundException ex) {        }        Method m;        try {            //得到 SystemServer 的 main 方法            m = cl.getMethod("main", new Class[] { String[].class });        } catch (NoSuchMethodException ex) {        }        int modifiers = m.getModifiers();        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {        }        /*         * This throw gets caught in ZygoteInit.main(), which responds         * by invoking the exception's run() method. This arrangement         * clears up all the stack frames that were required in setting         * up the process.         */        throw new ZygoteInit.MethodAndArgsCaller(m, argv);    }
```

- className 为 com.android.server.SystemServer，通过反射得到SystemServer的class
- 通过 class 得到SystemServer的 main 方法
- 将 main 方法传入 MethodAndArgsCaller 异常中并抛出该异常。该异常会在ZygoteInit的 main方法中捕获。最终反射执行SystemServer.main方法

#### 4.5、SystemServer.main()

```java
    public static void main(String[] args) {        new SystemServer().run();    }    private void run() {        try {            // Here we go!            Slog.i(TAG, "Entered the Android system server!");            int uptimeMillis = (int) SystemClock.elapsedRealtime();            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, uptimeMillis);            if (!mRuntimeRestart) {                MetricsLogger.histogram(null, "boot_system_server_init", uptimeMillis);            }			//变更虚拟机的库文件            SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());            // Mmmmmm... more memory!            //清除vm内存增长上限，因为启动过程中需要较多的虚拟机内存空间            VMRuntime.getRuntime().clearGrowthLimit();            // The system server has to run all of the time, so it needs to be            // as efficient as possible with its memory usage.            //设置内存有效使用率为0.8            VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);            //访问环境路径前，显示指定用户            BaseBundle.setShouldDefuse(true);            // Ensure binder calls into the system always run at foreground priority.            //确保对系统的Binder调用总是以前台优先级运行。            BinderInternal.disableBackgroundScheduling(true);            // Increase the number of binder threads in system_server            //增加system_server中绑定器线程的数量            BinderInternal.setMaxThreads(sMaxBinderThreads);            // Prepare the main looper thread (this thread).            Process.setThreadPriority(                Process.THREAD_PRIORITY_FOREGROUND);            Process.setCanSelfBackground(false);            //创建当前线程 的Looper            Looper.prepareMainLooper();            // Initialize native services.            //加载android_servers.so库            System.loadLibrary("android_servers");            // Check whether we failed to shut down last time we tried.            // This call may not return.            performPendingShutdown();            // Initialize the system context.            //创建系统Context            createSystemContext();            // Create the system service manager.            //创建系统服务管理器。            mSystemServiceManager = new SystemServiceManager(mSystemContext);            mSystemServiceManager.setRuntimeRestarted(mRuntimeRestart);            //将 mSystemServiceManager 添加到管理本地服务的sLocalServiceObjects map中            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);            // Prepare the thread pool for init tasks that can be parallelized            SystemServerInitThreadPool.get();        } finally {            traceEnd();  // InitBeforeStartServices        }        // Start services.        //启动各种服务        try {            traceBeginAndSlog("StartServices");            startBootstrapServices();//启动引导服务            startCoreServices();//启动核心服务            startOtherServices();//启动其他服务            SystemServerInitThreadPool.shutdown();        } catch (Throwable ex) {            throw ex;        } finally {            traceEnd();        }        // Loop forever.        Looper.loop();        throw new RuntimeException("Main thread loop unexpectedly exited");    }
```

在run方法中主要做了：

- 创建主线程的`Looper`对象
- 加载了动态库`android_servers.so`
- 创建了全局`Context`
- 创建`SystemServiceManager`对象，用于对系统服务的创建、启动和声明周期管理
- 启动各种服务

#### 4.6、总结

SystemServer是Zygote进程fork创建的第一个进程，主要做了：

- 启动了Binder线程池，这样就可以与其他进程进行通信了
- 创建SystemServerManager，用于对系统服务进行创建、启动和声明周期管理
- 启动各种服务

### 五、Android进程启动

每个`App`在启动前必须先创建一个进程，该进程是由`Zygote` fork出来的，进程具有独立的资源空间，用于承载App上运行的各种Activity/Service等组件。

- `system_server`进程：是用于管理整个Java framework层，包含ActivityManager，PowerManager等各种系统服务;
- `Zygote`进程：是Android系统的首个Java进程，Zygote是所有Java进程的父进程，包括 `system_server`进程以及所有的App进程都是Zygote的子进程，注意这里说的是子进程，而非子线程。

**点击桌面应用进程创建流程：**

1. **App发起进程**：当从桌面启动应用，则发起进程便是Launcher所在进程；当从某App内启动远程进程，则发送进程便是该App所在进程。发起进程先通过binder发送消息给system_server进程；
2. **system_server进程**：调用Process.start()方法，通过socket向zygote进程发送创建新进程的请求；
3. **zygote进程**：在执行`ZygoteInit.main()`后便进入`runSelectLoop()`循环体内，当有客户端连接时便会执行ZygoteConnection.runOnce()方法，再经过层层调用后fork出新的应用进程；
4. **新进程**：执行handleChildProc方法，最后调用ActivityThread.main()方法。

#### 1、AMS发起创建进程请求

##### 1.1、Process.Start()

```java
public static final ProcessStartResult start(final String processClass, final String niceName, int uid, int gid, int[] gids, int debugFlags, int mountExternal, int targetSdkVersion, String seInfo, String abi, String instructionSet, String appDataDir, String[] zygoteArgs) {    try {        return startViaZygote(processClass, niceName, uid, gid, gids,                debugFlags, mountExternal, targetSdkVersion, seInfo,                abi, instructionSet, appDataDir, zygoteArgs);    } catch (ZygoteStartFailedEx ex) {        throw new RuntimeException("");    }}...
```

核心参数：processClass：ActivityThread全类名、进程名、uid、gid 

通过socket向Zygote进程发送创建进程消息，这是便会唤醒Zygote进程，来响应socket客户端的请求。

#### 2、Zygote创建进程

##### 2.1、ZygoteInit.main

```java
public static void main(String argv[]) {    try {        runSelectLoop(abiList); // 轮询        ....    } catch (MethodAndArgsCaller caller) {        caller.run(); // 执行反射方法    } catch (RuntimeException ex) {        closeServerSocket();        throw ex;    }}
```

##### 2.2、runSelectLoop

该方法主要功能：

- 客户端通过openZygoteSocketIfNeeded()来跟zygote进程建立连接。zygote进程收到客户端连接请求后执行accept()；然后再创建ZygoteConnection对象,并添加到fds数组列表；
- 建立连接之后，可以跟客户端通信，进入runOnce()方法来接收客户端数据，并执行进程创建工作。

##### 2.3、**ZygoteConnection & `runOnce()`**

```java
...        try {        	//解析启动参数            parsedArgs = new Arguments(args);  			...			//启动进程            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,                    parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,                    parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,                    parsedArgs.appDataDir);        } catch (ErrnoException ex) {            ...        }         try {            if (pid == 0) {                // in child                IoUtils.closeQuietly(serverPipeFd);                serverPipeFd = null;                //处理应用程序进程                handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);                return true;            } else {                // in parent...pid of < 0 means failure                IoUtils.closeQuietly(childPipeFd);                childPipeFd = null;                return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);            }        } finally {        }
```

#### 3、新进程运行

##### 3.1、handleChildProc

```java
    private void handleChildProc(Arguments parsedArgs,            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)            throws ZygoteInit.MethodAndArgsCaller {		//关闭socket        closeSocket();        ZygoteInit.closeServerSocket();		...		//设置进程名        if (parsedArgs.niceName != null) {            Process.setArgV0(parsedArgs.niceName);        }        if (parsedArgs.invokeWith != null) {            WrapperInit.execApplication(parsedArgs.invokeWith,                    parsedArgs.niceName, parsedArgs.targetSdkVersion,                    VMRuntime.getCurrentInstructionSet(),                    pipeFd, parsedArgs.remainingArgs);        } else {            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,                    parsedArgs.remainingArgs, null /* classLoader */);        }    }
```

##### 3.2、**RuntimeInit**.zygoteInit()

```java
    public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)            throws ZygoteInit.MethodAndArgsCaller {        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");        redirectLogStreams();        commonInit();        //创建Binder线程池        nativeZygoteInit();        applicationInit(targetSdkVersion, argv, classLoader);    }
```

**`applicationInit`**

```java
    private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)            throws ZygoteInit.MethodAndArgsCaller {        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);        final Arguments args;        try {            args = new Arguments(argv);        } catch (IllegalArgumentException ex) {            return;        }        // Remaining arguments are passed to the start class's static main        invokeStaticMain(args.startClass, args.startArgs, classLoader);    }
```

**invokeStaticMain**

```java
    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)            throws ZygoteInit.MethodAndArgsCaller {        Class<?> cl;        try {        	//通过className获得ActivityThread的Class对象        	//className 为 android.app.ActivityThread            cl = Class.forName(className, true, classLoader);        } catch (ClassNotFoundException ex) {        }        Method m;        try {        	//通过反射获得main方法            m = cl.getMethod("main", new Class[] { String[].class });        } catch (NoSuchMethodException ex) {        }        int modifiers = m.getModifiers();        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {        }        /*         * This throw gets caught in ZygoteInit.main(), which responds         * by invoking the exception's run() method. This arrangement         * clears up all the stack frames that were required in setting         * up the process.         */        throw new ZygoteInit.MethodAndArgsCaller(m, argv);    }
```

- 该方法会通过反射获得`ActivityThread`的main方法，并将它传递给抛出的`ZygoteInit.MethodAndArgsCaller`异常中
- 在ZygoteInit.main中捕获MethodAndArgsCaller异常，并执行run方法，在run方法中反射执行ActivityThread的main方法

##### 3.3、ActivityThread.main

```java
    public static void main(String[] args) {		...		//创建主线程的Looper        Looper.prepareMainLooper();		//创建ActivityThread        ActivityThread thread = new ActivityThread();        thread.attach(false);		//创建主线程Handler        if (sMainThreadHandler == null) {            sMainThreadHandler = thread.getHandler();        }        // End of event ActivityThreadMain.        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);        //循环工作        Looper.loop();        throw new RuntimeException("Main thread loop unexpectedly exited");    }
```

可以看到，在`ActivityThread` 的 `main()` 方法中会创建消息循环所需要的`Handler`、并且绑定主线程的`Looper`并启动。这样就可以处理消息了。
