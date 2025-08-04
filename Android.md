# Android

## APP启动流程梳理

### 1.什么是Zygote进程
#### 1.1 简单介绍
- Zygote进程是所有的android进程的父进程，包括SystemServer和各种应用进程都是通过Zygote进程fork出来的。Zygote（孵化）进程相当于是android系统的根进程，后面所有的进程都是通过这个进程fork出来的
- 虽然Zygote进程相当于Android系统的根进程，但是事实上它也是由Linux系统的init进程启动的。


#### 1.2 各个进程的先后顺序
- init进程 --> Zygote进程 --> SystemServer进程 -->各种应用进程


#### 1.3 进程作用说明
- init进程：linux的根进程，android系统是基于linux系统的，因此可以算作是整个android操作系统的第一个进程；
- Zygote进程：android系统的根进程，主要作用：可以作用Zygote进程fork出SystemServer进程和各种应用进程；
- SystemService进程：主要是在这个进程中启动系统的各项服务，比如ActivityManagerService，PackageManagerService，WindowManagerService服务等等；
- 各种应用进程：启动自己编写的客户端应用时，一般都是重新启动一个应用进程，有自己的虚拟机与运行环境；



### 2.Zygote进程的启动流程
#### 2.1 源码位置
- 位置：frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
- Zygote进程mian方法主要执行逻辑：
    - 初始化DDMS；
    - 注册Zygote进程的socket通讯；
    - 初始化Zygote中的各种类，资源文件，OpenGL，类库，Text资源等等；
    - 初始化完成之后fork出SystemServer进程；
    - fork出SystemServer进程之后，关闭socket连接；


#### 2.2 ZygoteInit类的main方法
- init进程在启动Zygote进程时一般都会调用ZygoteInit类的main方法，因此这里看一下该方法的具体实现(基于android23源码)；
    - 调用enableDdms()，设置DDMS可用，可以发现DDMS启动的时机还是比较早的，在整个Zygote进程刚刚开始要启动额时候就设置可用。
    - 之后初始化各种参数
    - 通过调用registerZygoteSocket方法，注册为Zygote进程注册Socket
    - 然后调用preload方法实现预加载各种资源
    - 然后通过调用startSystemServer开启SystemServer服务，这个是重点


#### 2.3 registerZygoteSocket(socketName)分析
- 调用registerZygoteSocket（String socketName）为Zygote进程注册socket

#### 2.4 preLoad()方法分析
- 源码如下所示
    ```
    static void preload() {
        Log.d(TAG, "begin preload");
        preloadClasses();
        preloadResources();
        preloadOpenGL();
        preloadSharedLibraries();
        preloadTextResources();
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        WebViewFactory.prepareWebViewInZygote();
        Log.d(TAG, "end preload");
    }
    ```
- 大概操作是这样的：
    - preloadClasses()用于初始化Zygote中需要的class类；
    - preloadResources()用于初始化系统资源；
    - preloadOpenGL()用于初始化OpenGL；
    - preloadSharedLibraries()用于初始化系统libraries；
    - preloadTextResources()用于初始化文字资源；
    - prepareWebViewInZygote()用于初始化webview;

#### 2.5 startSystemServer()启动进程
- 这段逻辑的执行逻辑就是通过Zygote fork出SystemServer进程
    ```
    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_BLOCK_SUSPEND,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_RESOURCE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG
        );
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;
    
        int pid;
    
        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
    
            /* Request to fork the system server process */
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
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }
    
            handleSystemServerProcess(parsedArgs);
        }
    
        return true;
    }
    ```


### 3.SystemServer进程启动流程
#### 3.1 SystemServer进程简介
- SystemServer进程主要的作用是在这个进程中启动各种系统服务，比如ActivityManagerService，PackageManagerService，WindowManagerService服务，以及各种系统性的服务其实都是在SystemServer进程中启动的，而当我们的应用需要使用各种系统服务的时候其实也是通过与SystemServer进程通讯获取各种服务对象的句柄的。


#### 3.2 SystemServer的main方法
- 如下所示，比较简单，只是new出一个SystemServer对象并执行其run方法，查看SystemServer类的定义我们知道其实final类型的，所以我们一般不能重写或者继承。
- ![image](https://upload-images.jianshu.io/upload_images/4432347-92aba0bb0f4a31e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 3.3 查看run方法
- 代码如下所示
    - 首先判断系统当前时间，若当前时间小于1970年1月1日，则一些初始化操作可能会处所，所以当系统的当前时间小于1970年1月1日的时候，设置系统当前时间为该时间点。
    - 然后是设置系统的语言环境等
    - 接着设置虚拟机运行内存，加载运行库，设置SystemServer的异步消息


#### 3.4 run方法中createSystemContext()解析
- 调用createSystemContext()方法：   
    - 可以看到在SystemServer进程中也存在着Context对象，并且是通过ActivityThread.systemMain方法创建context的，这一部分的逻辑以后会通过介绍Activity的启动流程来介绍，这里就不在扩展，只知道在SystemServer进程中也需要创建Context对象。
    ```
    private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
    }
    ```

#### 3.5 mSystemServiceManager的创建
- 看run方法中，通过SystemServiceManager的构造方法创建了一个新的SystemServiceManager对象，我们知道SystemServer进程主要是用来构建系统各种service服务的，而SystemServiceManager就是这些服务的管理对象。
- 然后调用：
    - 将SystemServiceManager对象保存SystemServer进程中的一个数据结构中。
    ```
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
    ```
- 最后开始执行：
    ```
    // Start services.
    try {
        startBootstrapServices();
        startCoreServices();
        startOtherServices();
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    }
    ```
    - 里面主要涉及了是三个方法：
        - startBootstrapServices() 主要用于启动系统Boot级服务
        - startCoreServices() 主要用于启动系统核心的服务
        - startOtherServices() 主要用于启动一些非紧要或者是非需要及时启动的服务


### 4.启动服务
#### 4.1 启动哪些服务
- 在开始执行启动服务之前总是会先尝试通过socket方式连接Zygote进程，在成功连接之后才会开始启动其他服务。 
- ![image](https://upload-images.jianshu.io/upload_images/4432347-a10553435289e229.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 4.2 启动服务流程源码分析
- 首先看一下startBootstrapServices方法：
    ```
    private void startBootstrapServices() {
        Installer installer = mSystemServiceManager.startService(Installer.class);
    
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
    
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
    
        mActivityManagerService.initPowerManagement();
    
        // Manages LEDs and display backlight so we need it to bring up the display.
        mSystemServiceManager.startService(LightsService.class);
    
        // Display manager is needed to provide display metrics before package manager
        // starts up.
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
    
        // We need the default display before we can initialize the package manager.
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
    
        // Only run "core" apps if we're encrypting the device.
        String cryptState = SystemProperties.get("vold.decrypt");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        }
    
        // Start the package manager.
        Slog.i(TAG, "Package Manager");
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();
    
        Slog.i(TAG, "User Service");
        ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());
    
        // Initialize attribute cache used to cache resources from packages.
        AttributeCache.init(mSystemContext);
    
        // Set up the Application instance for the system process and get started.
        mActivityManagerService.setSystemProcess();
    
        // The sensor service needs access to package manager service, app ops
        // service, and permissions service, therefore we start it after them.
        startSensorService();
    }
    ```
- 先执行：
    ```
    Installer installer = mSystemServiceManager.startService(Installer.class);
    ```
- mSystemServiceManager是系统服务管理对象，在main方法中已经创建完成，这里我们看一下其startService方法的具体实现：
    - 可以看到通过反射器构造方法创建出服务类，然后添加到SystemServiceManager的服务列表数据中，最后调用了service.onStart()方法，因为传递的是Installer.class 
    ```
    public <T extends SystemService> T startService(Class<T> serviceClass) {
        final String name = serviceClass.getName();
        Slog.i(TAG, "Starting " + name);
    
        // Create the service.
        if (!SystemService.class.isAssignableFrom(serviceClass)) {
            throw new RuntimeException("Failed to create " + name
                    + ": service must extend " + SystemService.class.getName());
        }
        final T service;
        try {
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            service = constructor.newInstance(mContext);
        } catch (InstantiationException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (InvocationTargetException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service constructor threw an exception", ex);
        }
    
        // Register it.
        mServices.add(service);
    
        // Start it.
        try {
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + name
                    + ": onStart threw an exception", ex);
        }
        return service;
    }
    ```
- 看一下Installer的onStart方法：
    - 很简单就是执行了mInstaller的waitForConnection方法，这里简单介绍一下Installer类，该类是系统安装apk时的一个服务类，继承SystemService（系统服务的一个抽象接口），需要在启动完成Installer服务之后才能启动其他的系统服务。 
    ```
    @Override
    public void onStart() {
        Slog.i(TAG, "Waiting for installd to be ready.");
        mInstaller.waitForConnection();
    }
    ```
- 然后查看waitForConnection（）方法：
    - 通过追踪代码可以发现，其在不断的通过ping命令连接Zygote进程（SystemServer和Zygote进程通过socket方式通讯，其他进程通过Binder方式通讯） 
    ```
    public void waitForConnection() {
        for (;;) {
            if (execute("ping") >= 0) {
                return;
            }
            Slog.w(TAG, "installd not ready");
            SystemClock.sleep(1000);
        }
    }
    ```
- 继续看startBootstrapServices方法：
    - 这段代码主要是用于启动ActivityManagerService服务，并为其设置SysServiceManager和Installer。ActivityManagerService是系统中一个非常重要的服务，Activity，service，Broadcast，contentProvider都需要通过其余系统交互。 
    ```
    // Activity manager runs the show.
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    ```
- 首先看一下Lifecycle类的定义：
    - 可以看到其实ActivityManagerService的一个静态内部类，在其构造方法中会创建一个ActivityManagerService，通过刚刚对Installer服务的分析我们知道，SystemServiceManager的startService方法会调用服务的onStart()方法，而在Lifecycle类的定义中我们看到其onStart（）方法直接调用了mService.start()方法，mService是Lifecycle类中对ActivityManagerService的引用
    ```
    public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;
    
        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        }
    
        @Override
        public void onStart() {
            mService.start();
        }
    
        public ActivityManagerService getService() {
            return mService;
        }
    }
    ```


#### 4.3 启动部分服务
- 启动PowerManagerService服务：
    - 启动方式跟上面的ActivityManagerService服务相似都会调用其构造方法和onStart方法，PowerManagerService主要用于计算系统中和Power相关的计算，然后决策系统应该如何反应。同时协调Power如何与系统其它模块的交互，比如没有用户活动时，屏幕变暗等等。
    ```
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
    ```
    - 然后是启动LightsService服务
        - 主要是手机中关于闪光灯，LED等相关的服务；也是会调用LightsService的构造方法和onStart方法；
        ```
        mSystemServiceManager.startService(LightsService.class);
        ```
    - 然后是启动DisplayManagerService服务
        - 主要是手机显示方面的服务
        ```
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
    
    ```
- **然后是启动PackageManagerService，该服务也是android系统中一个比较重要的服务**
    - 包括多apk文件的安装，解析，删除，卸载等等操作。
    - 可以看到PackageManagerService服务的启动方式与其他服务的启动方式有一些区别，直接调用了PackageManagerService的静态main方法
    ```
    Slog.i(TAG, "Package Manager");
    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager();
    ```
    - 看一下其main方法的具体实现：
        - 可以看到也是直接使用new的方式创建了一个PackageManagerService对象，并在其构造方法中初始化相关变量，最后调用了ServiceManager.addService方法，主要是通过Binder机制与JNI层交互
    ```
    public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        ServiceManager.addService("package", m);
        return m;
    }
    ```
- 然后查看startCoreServices方法：
    - 可以看到这里启动了BatteryService（电池相关服务），UsageStatsService，WebViewUpdateService服务等。
    ```
    private void startCoreServices() {
        // Tracks the battery level.  Requires LightService.
        mSystemServiceManager.startService(BatteryService.class);
    
        // Tracks application usage stats.
        mSystemServiceManager.startService(UsageStatsService.class);
        mActivityManagerService.setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));
        // Update after UsageStatsService is available, needed before performBootDexOpt.
        mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();
    
        // Tracks whether the updatable WebView is in a ready state and watches for update installs.
        mSystemServiceManager.startService(WebViewUpdateService.class);
    }
    ```

#### 总结：
- SystemServer进程是android中一个很重要的进程由Zygote进程启动；
- SystemServer进程主要用于启动系统中的服务；
- SystemServer进程启动服务的启动函数为main函数；
- SystemServer在执行过程中首先会初始化一些系统变量，加载类库，创建Context对象，创建SystemServiceManager对象等之后才开始启动系统服务；
- SystemServer进程将系统服务分为三类：boot服务，core服务和other服务，并逐步启动
- SertemServer进程在尝试启动服务之前会首先尝试与Zygote建立socket通讯，只有通讯成功之后才会开始尝试启动服务；
- 创建的系统服务过程中主要通过SystemServiceManager对象来管理，通过调用服务对象的构造方法和onStart方法初始化服务的相关变量；
- 服务对象都有自己的异步消息对象，并运行在单独的线程中；







## ActivityThread分析


### 01.ActivityThread介绍
- 整个Android应用进程的体系非常复杂，而ActivityThread是真正的核心类，它的main方法，是整个应用进程的入口。
- 接下来，就来了解一下，如何走到ActivityThread的main方法，然后在main方法中又做了什么……也就是从来里来，到哪里去？


### 02.启动Activity所属的应用进程
- 大概流程如下所示
    >ActivityManagerService.startProcessLocked()
    >Process.start()
    >ActivityThread.main()
    >ActivityThread.attach()
    >ActivityManagerNative.getDefault().attachApplication()
    >ActivityManagerService.attachApplication()
- 首先看一下startProcessLocked()方法的具体实现：
    ``` java
    private final void startProcessLocked(ProcessRecord app,
            String hostingType, String hostingNameStr) {
        startProcessLocked(app, hostingType, hostingNameStr, null /* abiOverride */,
                null /* entryPoint */, null /* entryPointArgs */);
    }
    ```
- 然后回调了其重载的startProcessLocked方法：
    ``` java
    private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
            ...
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
            ...
    }
    ```
- 可以发现其经过一系列的初始化操作之后调用了Process.start方法，并且传入了启动的类名“android.app.ActivityThread”:
    ``` java
    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }
    ```
- 然后调用了startViaZygote方法：
    ```
    private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        synchronized(Process.class) {
            ...
    
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
    ```
- 继续查看一下zygoteSendArgsAndGetResult方法的实现：
    ```
    private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            /**
             * See com.android.internal.os.ZygoteInit.readArgumentList()
             * Presently the wire format to the zygote process is:
             * a) a count of arguments (argc, in essence)
             * b) a number of newline-separated argument strings equal to count
             *
             * After the zygote process reads these it will write the pid of
             * the child or -1 on failure, followed by boolean to
             * indicate whether a wrapper process was used.
             */
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;
    
            writer.write(Integer.toString(args.size()));
            writer.newLine();
    
            int sz = args.size();
            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                if (arg.indexOf('\n') >= 0) {
                    throw new ZygoteStartFailedEx(
                            "embedded newlines not allowed");
                }
                writer.write(arg);
                writer.newLine();
            }
    
            writer.flush();
    
            // Should there be a timeout on this?
            ProcessStartResult result = new ProcessStartResult();
            result.pid = inputStream.readInt();
            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            result.usingWrapper = inputStream.readBoolean();
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }
    ```
- 可以发现其最终调用了Zygote并通过socket通信的方式让Zygote进程fork除了一个新的进程，并根据我们刚刚传递的"android.app.ActivityThread"字符串，反射出该对象并执行ActivityThread的main方法。这样我们所要启动的应用进程这时候其实已经启动了，但是还没有执行相应的初始化操作。



### 03.ActivityThread的main方法
- 为什么我们平时都将ActivityThread称之为ui线程或者是主线程
    - 这里可以看出，应用进程被创建之后首先执行的是ActivityThread的main方法，所以我们将ActivityThread成为主线程。
- 这时候我们看一下ActivityThread的main方法的实现逻辑。
    ```
    public static void main(String[] args) {
        ...
        Process.setArgV0("<pre-initialized>");
        //初始化Looper
        Looper.prepareMainLooper();
        //先创建ActivityThread对象
        ActivityThread thread = new ActivityThread();
        //attach
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
        //开启loop循环
        Looper.loop();
    
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
    ```
    - 在main方法中主要执行了一些初始化的逻辑，并且创建了一个UI线程消息队列，这也就是为什么我们可以在主线程中随意的创建Handler而不会报错的原因，这里提出一个问题，大家可以思考一下：子线程可以创建Handler么？可以的话应该怎么做？
- 然后执行了ActivityThread的attach方法，这里我们看一下attach方法执行了那些逻辑操作。
    ```
    private void attach(boolean system) {
    	...
    	final IActivityManager mgr = ActivityManagerNative.getDefault();
                try {
                    mgr.attachApplication(mAppThread);
                } catch (RemoteException ex) {
                    // Ignore
                }
    	...
    }
    ```
- 刚刚已经分析过ActivityManagerNative是ActivityManagerService的Binder client，所以这里调用了attachApplication实际上就是通过Binder机制调用了ActivityManagerService的attachApplication，具体调用的过程，我们看一下ActivityManagerService是如何实现的：
    ```
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
- 可以发现其回调了attachApplicationLocked方法，我们看一下这个方法的实现逻辑。这个在ActivityManagerService类中。
    ``` java
     @GuardedBy("this")
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
        ......
        mStackSupervisor.getActivityMetricsLogger().notifyBindApplication(app);
            if (app.isolatedEntryPoint != null) {
                // This is an isolated process which should just call an entry point instead of
                // being bound to an application.
                thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
            } else if (app.instr != null) {
                thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            }
        ......
    }
    ```
    - 该方法执行了一系列的初始化操作，这样我们整个应用进程已经启动起来了。具体还是要看一下bindApplication这个方法。
- 进去上面的方法看一下
    - thread.bindApplication() 
    - ActivityThread.ApplicationThread # bindApplication()
    ``` java
    public final void bindApplication(String processName, ApplicationInfo appInfo,
            List<ProviderInfo> providers, ComponentName instrumentationName,
            ProfilerInfo profilerInfo, Bundle instrumentationArgs,
            IInstrumentationWatcher instrumentationWatcher,
            IUiAutomationConnection instrumentationUiConnection, int debugMode,
            boolean enableBinderTracking, boolean trackAllocation,
            boolean isRestrictedBackupMode, boolean persistent, Configuration config,
            CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
            String buildSerial, boolean autofillCompatibilityEnabled) {
        
        if (services != null) {
            if (false) {
                // Test code to make sure the app could see the passed-in services.
                for (Object oname : services.keySet()) {
                    if (services.get(oname) == null) {
                        continue; // AM just passed in a null service.
                    }
                    String name = (String) oname;
        
                    // See b/79378449 about the following exemption.
                    switch (name) {
                        case "package":
                        case Context.WINDOW_SERVICE:
                            continue;
                    }
        
                    if (ServiceManager.getService(name) == null) {
                        Log.wtf(TAG, "Service " + name + " should be accessible by this app");
                    }
                }
            }
        
            // Setup the service cache in the ServiceManager
            ServiceManager.initServiceCache(services);
        }
        
        setCoreSettings(coreSettings);
        
        AppBindData data = new AppBindData();
        data.processName = processName;
        data.appInfo = appInfo;
        data.providers = providers;
        data.instrumentationName = instrumentationName;
        data.instrumentationArgs = instrumentationArgs;
        data.instrumentationWatcher = instrumentationWatcher;
        data.instrumentationUiAutomationConnection = instrumentationUiConnection;
        data.debugMode = debugMode;
        data.enableBinderTracking = enableBinderTracking;
        data.trackAllocation = trackAllocation;
        data.restrictedBackupMode = isRestrictedBackupMode;
        data.persistent = persistent;
        data.config = config;
        data.compatInfo = compatInfo;
        data.initProfilerInfo = profilerInfo;
        data.buildSerial = buildSerial;
        data.autofillCompatibilityEnabled = autofillCompatibilityEnabled;
        //主要看这里的代码
        sendMessage(H.BIND_APPLICATION, data);
    }
    ```
    - 然后看一下哪里处理消息。可以看到调用了handleBindApplication方法。
    ``` java
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case BIND_APPLICATION:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                AppBindData data = (AppBindData)msg.obj;
                handleBindApplication(data);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
    ```
- 接着看一下handleBindApplication方法做了什么
    - ActivityThread中的handleBindApplication
    ``` java
    private void handleBindApplication(AppBindData data) {
        //设置进程名字
        Process.setArgV0(data.processName);
        android.ddm.DdmHandleAppName.setAppName(data.processName,
                                                UserHandle.myUserId());
        VMRuntime.setProcessPackageName(data.appInfo.packageName);
    
        if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
            AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
        }
        // 创建ContextImpl
        final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
        Application app;
        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
        final StrictMode.ThreadPolicy writesAllowedPolicy = StrictMode.getThreadPolicy();
        try {
            // data.info的对象是LoadedApk, 通过反射创建目标应用Application对象
            app = data.info.makeApplication(data.restrictedBackupMode, null);
            app.setAutofillCompatibilityEnabled(data.autofillCompatibilityEnabled);
            mInitialApplication = app;
                mInstrumentation.onCreate(data.instrumentationArgs);
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
               
            }
        } finally {
        }
    }
    ```
- 跟踪到mInstrumentation.callApplicationOnCreate(app);
    - 可以看到Application进入onCreate()方法中，这就是从最开始main()方法开始到最后的Application的onCreate（）的创建过程。
    ``` java
    public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }
    ```

## Activity基础介绍

### 01.Activity生命周期
#### 1.1 生命周期方法说明
- 在正常情况下，一个Activity从启动到结束会以如下顺序经历整个生命周期：  
    - (1) onCreate()：当 Activity 第一次创建时会被调用。这是生命周期的第一个方法。在这个方法中，可以做一些初始化工作，比如调用setContentView去加载界面布局资源，初始化Activity所需的数据。当然也可借助onCreate()方法中的Bundle对象来回复异常情况下Activity结束时的状态（后面会介绍）。
    - (2) onRestart()：表示Activity正在重新启动。一般情况下，当当前Activity从不可见重新变为可见状态时，onRestart就会被调用。这种情形一般是用户行为导致的，比如用户按Home键切换到桌面或打开了另一个新的Activity，接着用户又回到了这个Actvity。（关于这部分生命周期的历经过程，后面会介绍。）
    - (3) onStart(): 表示Activity正在被启动，即将开始，这时Activity已经**出现**了，但是还没有出现在前台，无法与用户交互。这个时候可以理解为**Activity已经显示出来，但是我们还看不到。**
    - (4) onResume():表示Activity**已经可见了，并且出现在前台并开始活动**。需要和onStart()对比，onStart的时候Activity还在后台，onResume的时候Activity才显示到前台。
    - (5) onPause():表示 Activity正在停止，仍可见，正常情况下，紧接着onStop就会被调用。在特殊情况下，如果这个时候快速地回到当前Activity，那么onResume就会被调用（极端情况）。**onPause中不能进行耗时操作，会影响到新Activity的显示。因为onPause必须执行完，新的Activity的onResume才会执行。**
    - (6) onStop():表示Activity即将停止，不可见，位于后台。可以做稍微重量级的回收工作，同样不能太耗时。
    - (7) onDestroy():表示Activity即将销毁，这是Activity生命周期的最后一个回调，可以做一些回收工作和最终的资源回收。
- 在平常的开发中，我们经常用到的就是 onCreate()和onDestroy()，做一些初始化和回收操作。

#### 1.2 特殊情况下生命周期
- 第一种情况，销毁当前的Activity后重建，这种也尽量避免。
    - 在横竖屏切换的过程中，会发生Activity被销毁并重建的过程。在了解这种情况下的生命周期时，首先应该了解这两个回调：**onSaveInstanceState和onRestoreInstanceState**。
    - 在Activity由于异常情况下终止时，系统会调用onSaveInstanceState来保存当前Activity的状态。这个方法的调用是在onStop之前，它和onPause没有既定的时序关系，该方法只在Activity被异常终止的情况下调用。
    - 当异常终止的Activity被重建以后，系统会调用onRestoreInstanceState，并且把Activity销毁时onSaveInstanceState方法所保存的Bundle对象参数同时传递给onRestoreInstanceState和onCreate方法。
    - 因此，可以通过onRestoreInstanceState方法来恢复Activity的状态，该方法的调用时机是在onStart之后。**其中onCreate和onRestoreInstanceState方法来恢复Activity的状态的区别：** onRestoreInstanceState回调则表明其中Bundle对象非空，不用加非空判断。onCreate需要非空判断。建议使用onRestoreInstanceState。  
- 第二种情况，当前的Activity不销毁，设置Activity的属性。
    - 比如视频播放器就经常会涉及屏幕旋转场景。可以通过在AndroidManifest文件的Activity中指定如下属性：
    ``` java
    <activity
        android:name=".activity.VideoDetailActivity"
        android:configChanges="orientation|keyboardHidden|screenSize"
        android:screenOrientation="portrait"/>
    ```
    - 来避免横竖屏切换时，Activity的销毁和重建，而是回调了下面的方法：
    ``` java
    //重写旋转时方法，不销毁activity
    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
    }
    ```
- 资源内存不足导致优先级低的Activity被杀死
    - Activity优先级的划分和下面的Activity的三种运行状态是对应的。
    - (1) 前台Activity——正在和用户交互的Activity，优先级最高。
    - (2) 可见但非前台Activity——比如Activity中弹出了一个对话框，导致Activity可见但是位于后台无法和用户交互。
    - (3) 后台Activity——已经被暂停的Activity，比如执行了onStop，优先级最低。
    - 当系统内存不足时，会按照上述优先级从低到高去杀死目标Activity所在的进程。我们在平常使用手机时，能经常感受到这一现象。这种情况下数组存储和恢复过程和上述情况一致，生命周期情况也一样。



#### 1.3 Activity三种运行状态
- ①Resumed（活动状态）
    - 又叫Running状态，这个Activity正在屏幕上显示，并且有用户焦点。这个很好理解，就是用户正在操作的那个界面。
- ②Paused（暂停状态）
    - 这是一个比较不常见的状态。这个Activity在屏幕上是可见的，但是并不是在屏幕最前端的那个Activity。比如有另一个非全屏或者透明的Activity是Resumed状态，没有完全遮盖这个Activity。
- ③Stopped（停止状态）
    - 当Activity完全不可见时，此时Activity还在后台运行，仍然在内存中保留Activity的状态，并不是完全销毁。这个也很好理解，当跳转的另外一个界面，之前的界面还在后台，按回退按钮还会恢复原来的状态，大部分软件在打开的时候，直接按Home键，并不会关闭它，此时的Activity就是Stopped状态。



#### 1.4 App切换到后台分析
- app切换到后台，当前activity会走onDestroy方法吗？
    - 不会走onDestroy方法，会先后走onPause和onStop方法。
- 一般在onStop方法里做什么？
    - 比如。写轮播图的时候，会在onStop方法里写上暂停轮播图无限轮播，在onStart方法中会开启自动无限轮播。
    - 再比如，写视频播放器的时候，当app切换到后台，则需要停止视频播放，也是可以在onStop中处理的。
- 什么情况会导致app会被杀死，这时候会走onDestroy吗？
    - 系统资源不足，会导致app意外被杀死。应用只有在进程存活的情况下才会按照正常的生命周期进行执行，如果进程突然被kill掉，相当于System.exit(0); 进程被杀死，根本不会走（activity，fragment）生命周期。只有在进程不被kill掉，正常情况下才会执行onDestroy（）方法。

### 02.Activity的启动模式
#### 2.1 启动模式的类别
- Android提供了四种Activity启动方式：
    - 标准模式（standard）  
    - 栈顶复用模式（singleTop）  
    - 栈内复用模式（singleTask）  
    - 单例模式（singleInstance）


#### 2.2 启动模式的结构——栈
- Activity的管理是采用任务栈的形式，任务栈采用“后进先出”的栈结构。  
    - ![](http://upload-images.jianshu.io/upload_images/3985563-83df2c2b5d3afafd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 2.3 标准模式(standard)
- 每启动一次Activity，就会创建一个新的Activity实例并置于栈顶。谁启动了这个Activity，那么这个Activity就运行在启动它的那个Activity所在的栈中。
    - 例如：Activity A启动了Activity B，则就会在A所在的栈顶压入一个新的Activity。  
    - ![](http://upload-images.jianshu.io/upload_images/3985563-59e0c33aadd55f62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 特殊场景
    - 特殊情况，如果在Service或Application中启动一个Activity，其并没有所谓的任务栈，可以使用标记位Flag来解决。
        - 解决办法：为待启动的Activity指定FLAG_ACTIVITY_NEW_TASK标记位，创建一个新栈。
    - **应用场景**： 
        - 绝大多数Activity。如果以这种方式启动的Activity被跨进程调用，在5.0之前新启动的Activity实例会放入发送Intent的Task的栈的顶部，尽管它们属于不同的程序，这似乎有点费解看起来也不是那么合理。
        - 所以在5.0之后，上述情景会创建一个新的Task，新启动的Activity就会放入刚创建的Task中，这样就合理的多了。


#### 2.4 栈顶复用模式(singleTop)
- 如果需要新建的Activity位于任务栈栈顶，那么此Activity的实例就不会重建，而是重用栈顶的实例。并回调如下方法：
    ``` java
    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
    }
    ```
    - 由于不会重建一个Activity实例，则不会回调其他生命周期方法。  
- 如果栈顶不是新建的Activity，就会创建该Activity新的实例，并放入栈顶。  
    - ![](http://upload-images.jianshu.io/upload_images/3985563-6378dae7ea249294.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- **应用场景：** 
    - 在通知栏点击收到的通知，然后需要启动一个Activity，这个Activity就可以用singleTop，否则每次点击都会新建一个Activity。
    - 当然实际的开发过程中，测试妹纸没准给你提过这样的bug：某个场景下连续快速点击，启动了两个Activity。如果这个时候待启动的Activity使用 singleTop模式也是可以避免这个Bug的。
    - 同standard模式，如果是外部程序启动singleTop的Activity，在Android 5.0之前新创建的Activity会位于调用者的Task中，5.0及以后会放入新的Task中。



#### 2.5 栈内复用模式(singleTask)
- 该模式是一种单例模式，即一个栈内只有一个该Activity实例。
    - 该模式，可以通过在AndroidManifest文件的Activity中指定该Activity需要加载到那个栈中，即singleTask的Activity可以指定想要加载的目标栈。
    - singleTask和taskAffinity配合使用，指定开启的Activity加入到哪个栈中。
    ``` xml
    <activity android:name=".Activity1"
    	android:launchMode="singleTask"
    	android:taskAffinity="com.yc.task"
    	android:label="@string/app_name">
    </activity>
    ```
- **关于taskAffinity的值：** 
  
    - 每个Activity都有taskAffinity属性，这个属性指出了它希望进入的Task。如果一个Activity没有显式的指明该Activity的taskAffinity，那么它的这个属性就等于Application指明的taskAffinity，如果Application也没有指明，那么该taskAffinity的值就等于包名。
    - 这个也可以不用设置
    **执行逻辑：**
    - 在这种模式下，如果Activity指定的栈不存在，则创建一个栈，并把创建的Activity压入栈内。
    - 如果Activity指定的栈存在，如果其中没有该Activity实例，则会创建Activity并压入栈顶，如果其中有该Activity实例，则把该Activity实例之上的Activity杀死清除出栈，重用并让该Activity实例处在栈顶，然后调用onNewIntent()方法。
    - 对应如下三种情况：  
    - ![](http://upload-images.jianshu.io/upload_images/3985563-2309a7222394cd80.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    - ![](http://upload-images.jianshu.io/upload_images/3985563-b4e3dbc29cdb7da4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    - ![](http://upload-images.jianshu.io/upload_images/3985563-4c1598adf8d53e88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- **应用场景：** 
    - 大多数App的主页。对于大部分应用，当我们在主界面点击回退按钮的时候都是退出应用，那么当我们第一次进入主界面之后，主界面位于栈底，以后不管我们打开了多少个Activity，只要我们再次回到主界面，都应该使用将主界面Activity上所有的Activity移除的方式来让主界面Activity处于栈顶，而不是往栈顶新加一个主界面Activity的实例，通过这种方式能够保证退出应用时所有的Activity都能报销毁。
    - 在跨应用Intent传递时，如果系统中不存在singleTask Activity的实例，那么将创建一个新的Task，然后创建SingleTask Activity的实例，将其放入新的Task中。
- 应用案例
    - 例如浏览器的主界面。不管从多少个应用启动浏览器，只会启动主界面一次，其余情况都会走onNewIntent，并且会清空主界面上面的其他页面。




#### 2.6 单例模式(singleInstance)
- 作为栈内复用模式（singleTask）的加强版
    - 打开该Activity时，直接创建一个新的任务栈，并创建该Activity实例放入新栈中。一旦该模式的Activity实例已经存在于某个栈中，任何应用再激活该Activity时都会重用该栈中的实例。  
    - ![](http://upload-images.jianshu.io/upload_images/3985563-19d7d46af3e7acf7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- **应用场景：** 
    - 呼叫来电界面。这种模式的使用情况比较罕见，在Launcher中可能使用。或者你确定你需要使Activity只有一个实例。建议谨慎使用。

#### 2.7 Activity的Flags
- Activity的Flags很多，这里介绍集中常用的，用于设定Activity的启动模式。可以在启动Activity时，通过Intent的addFlags()方法设置。
    - **(1)FLAG_ACTIVITY_NEW_TASK**
    - 其效果与指定Activity为singleTask模式一致。
    - **(2)FLAG_ACTIVITY_SINGLE_TOP**
    - 其效果与指定Activity为singleTop模式一致。
    - **(3)FLAG_ACTIVITY_CLEAR_TOP**
    - 具有此标记位的Activity，当它启动时，在同一个任务栈中所有位于它上面的Activity都要出栈。如果和singleTask模式一起出现，若被启动的Activity已经存在栈中，则清除其之上的Activity，并调用该Activity的onNewIntent方法。如果被启动的Activity采用standard模式，那么该Activity连同之上的所有Activity出栈，然后创建新的Activity实例并压入栈中。
- 同一程序不同的Activity是否可以放在不同的Task任务栈中？
    - 可以的。比如：启动模式里有个singleInstance，可以运行在另外的单独的任务栈里面。用这个模式启动的activity，在内存中只有一份，这样就不会重复的开启。
    - 也可以在激活一个新的activity时候，给intent设置flag，Intent的flag添加FLAG_ACTIVITY_NEW_TASK，这个被激活的activity就会在新的task栈里面



### 03.特殊情况栈交互
- 特殊情况——前台栈和后台栈的交互
    - 假如目前有两个任务栈。前台任务栈为AB，后台任务栈为CD，这里假设CD的启动模式均为singleTask,现在请求启动D，那么这个后台的任务栈都会被切换到前台，这个时候整个后退列表就变成了ABCD。当用户按back返回时，列表中的activity会一一出栈。
- 如下图。
    - ![](http://upload-images.jianshu.io/upload_images/3985563-4aeb1947bba27e44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 如果不是请求启动D而是启动C，那么情况又不一样，如下图。  
    - ![](http://upload-images.jianshu.io/upload_images/3985563-f2eaf1005cdf1b1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 调用SingleTask模式的后台任务栈中的Activity，会把整个栈的Activity压入当前栈的栈顶。
    - singleTask会具有clearTop特性，把之上的栈内Activity清除。

### 04.Activity异常情况
#### 4.1 Activity异常生命周期
- 虽然前面也说了这个异常的生命周期，但是还是有些疑问？
- 异常条件会调用什么方法
    - 当非人为终止Activity时，比如系统配置发生改变时导致Activity被杀死并重新创建、资源内存不足导致低优先级的Activity被杀死，会调用 onSavaInstanceState() 来保存状态。该方法调用在onStop之前，但和onPause没有时序关系。
    - 有人会问，onSaveInstanceState()与onPause()的区别，onSaveInstanceState()适用于对临时性状态的保存，而onPause()适用于对数据的持久化保存。
    - 当异常崩溃后App又重启了，这个时候会走onRestoreInstanceState()方法，可以在该方法中取出onSaveInstanceState()保存的状态数据。
- 什么时候会引起异常生命周期
    - 资源相关的系统配置发生改变或者资源不足：例如屏幕旋转，当前Activity会销毁，并且在onStop之前回调onSaveInstanceState保存数据，在重新创建Activity的时候在onStart之后回调onRestoreInstanceState。其中Bundle数据会传到onCreate（不一定有数据）和onRestoreInstanceState（一定有数据）。
    - 还有中情况是异常导致app崩溃，然后并没有完全杀死进程，重启回到activity页面，也会引起异常生命周期。


#### 4.2 后台Activity被异常回收
- 后台的Activity被系统回收怎么办？
    - Activity中提供了一个 onSaveInstanceState()回调方法，这个方法会保证一定在活动被回收之前调用，可以通过这个方法来解决活动被回收时临时数据得不到保存的问题。
    - onSaveInstanceState()方法会携带一个Bundle类型的参数，Bundle提供了一系列的方法用于保存数据，比如可以使用putString()方法保存字符串，使用putInt()方法保存整型数据。每个保存方法需要传入两个参数，第一个参数是键，用于后面从 Bundle中取值，第二个参数是真正要保存的内容。
- 说一下onSaveInstanceState()和onRestoreInstanceState()方法特点？
    - Activity的 onSaveInstanceState()和onRestoreInstanceState()并不是生命周期方法，它们不同于onCreate()、onPause()等生命周期方法，它们并不一定会被触发。
        ``` java
        //保存数据
        @Override
        protected void onSaveInstanceState(Bundle outBundle) {
        	super.onSaveInstanceState(outBundle);
         	outBundle.putBoolean("Change", mChange);
        }
        
        //取出数据
        @Override 
        protected void onRestoreInstanceState(Bundle savedInstanceState) {
        	super.onRestoreInstanceState(savedInstanceState);
        	mChange = savedInstanceState.getBoolean("Change");
        }
        
        //或者在onCreate方法取数据也可以
        //onCreate()方法其实也有一个Bundle类型的参数。这个参数在一般情况下都是null，
        //但是当活动被系统回收之前有通过 onSaveInstanceState()方法来保存数据的话，这个参就会带有之前所保存的全部数据
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            if (savedInstanceState != null) {
                String data = savedInstanceState.getString("data");
            }
        }
        ```
- 什么时候会触发走这两个方法？
    - 当应用遇到意外情况（如：内存不足、用户直接按Home键）由系统销毁一个Activity，onSaveInstanceState() 会被调用。但是当用户主动去销毁一个Activity时，例如在应用中按返回键，onSaveInstanceState()就不会被调用。除非该activity是被用户主动销毁的，通常onSaveInstanceState()只适合用于保存一些临时性的状态，而onPause()适合用于数据的持久化保存。
- onSaveInstanceState()被执行的场景有哪些？
    - 系统不知道你按下HOME后要运行多少其他的程序，自然也不知道activityA是否会被销毁，因此系统都会调用onSaveInstanceState()，让用户有机会保存某些非永久性的数据。以下几种情况的分析都遵循该原则当用户按下HOME键时
        - 长按HOME键，选择运行其他的程序时
        - 锁屏时
        - 从activity A中启动一个新的activity时
        - 屏幕方向切换时




- 屏幕旋转时生命周期
    - 屏幕旋转时候，如果不做任何处理，activity会经过销毁到重建的过程。一般这种效果都不是想要的。比如视频播放器就经常会涉及屏幕旋转场景。[技术博客大总结](https://github.com/yangchong211/YCBlogs)
    - 第一种情况：当前的Activity不销毁【设置Activity的android:configChanges="orientation|keyboardHidden|screenSize"时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged方法】
        ```
        <activity
            android:name=".activity.VideoDetailActivity"
            android:configChanges="orientation|keyboardHidden|screenSize"
            android:screenOrientation="portrait"/>
        ```
        - 执行该方法
        ```
        //重写旋转时方法，不销毁activity
        @Override
        public void onConfigurationChanged(Configuration newConfig) {
        	super.onConfigurationChanged(newConfig);
        }
        ```
    - 第二种情况：销毁当前的Activity后重建，这种也尽量避免。【不设置Activity的android:configChanges时，切屏会重新调用各个生命周期，默认首先销毁当前activity,然后重新加载】



### 1.0.0.3 如何避免配置改变时Activity重建？优先级低的Activity在内存不足被回收后怎样做可以恢复到销毁前状态？
- 如何避免配置改变时Activity重建
    - 为了避免由于配置改变导致Activity重建，可在AndroidManifest.xml中对应的Activity中设置android:configChanges="orientation|screenSize"。此时再次旋转屏幕时，该Activity不会被系统杀死和重建，只会调用onConfigurationChanged。因此，当配置程序需要响应配置改变，指定configChanges属性，重写onConfigurationChanged方法即可。
    - 使用场景，比如视频播放器横竖屏切换播放视频，就需要设置这种属性。具体可以看我封装的视频播放器库，地址：https://github.com/yangchong211/YCVideoPlayer
- 优先级低的Activity在内存不足被回收后怎样做可以恢复到销毁前状态
    - 优先级低的Activity在内存不足被回收后重新打开会引发Activity重建。Activity被重新创建时会调用onRestoreInstanceState（该方法在onStart之后），并将onSavaInstanceState保存的Bundle对象作为参数传到onRestoreInstanceState与onCreate方法。因此可通过onRestoreInstanceState(Bundle savedInstanceState)和onCreate((Bundle savedInstanceState)来判断Activity是否被重建，并取出数据进行恢复。但需要注意的是，在onCreate取出数据时一定要先判断savedInstanceState是否为空。
- 如何判断activity的优先级？[技术博客大总结](https://github.com/yangchong211/YCBlogs)
    - 除了在栈顶的activity,其他的activity都有可能在内存不足的时候被系统回收，一个activity越处于栈底，被回收的可能性越大.如果有多个后台进程，在选择杀死的目标时，采用最近最少使用算法（LRU）。


### 1.0.0.6 说下Activity的四种启动模式？singleTop和singleTask的区别以及应用场景？任务栈的作用是什么？
- Activity的四种启动模式
    - standard标准模式：每次启动一个Activity就会创建一个新的实例
    - singleTop栈顶复用模式：如果新Activity已经位于任务栈的栈顶，就不会重新创建，并回调 onNewIntent(intent) 方法
    - singleTask栈内复用模式：只要该Activity在一个任务栈中存在，都不会重新创建，并回调 onNewIntent(intent) 方法。如果不存在，系统会先寻找是否存在需要的栈，如果不存在该栈，就创建一个任务栈，并把该Activity放进去；如果存在，就会创建到已经存在的栈中
    - singleInstance单实例模式：具有此模式的Activity只能单独位于一个任务栈中，且此任务栈中只有唯一一个实例
- singleTop和singleTask的区别以及应用场景
    - singleTop：同个Activity实例在栈中可以有多个，即可能重复创建；该模式的Activity会默认进入启动它所属的任务栈，即不会引起任务栈的变更；为防止快速点击时多次startActivity，可以将目标Activity设置为singleTop
    - singleTask：同个Activity实例在栈中只有一个，即不存在重复创建；可通过android：taskAffinity设定该Activity需要的任务栈，即可能会引起任务栈的变更；常用于主页和登陆页
- singleTop或singleTask的Activity在以下情况会回调onNewIntent()
    - singleTop：如果新Activity已经位于任务栈的栈顶，就不会重新创建，并回调 onNewIntent(intent) 方法
    - singleTask：只要该Activity在一个任务栈中存在，都不会重新创建，并回调 onNewIntent(intent) 方法
- 任务栈的作用是什么？[技术博客大总结](https://github.com/yangchong211/YCBlogs)
    - 它是存放 Activity 的引用的，Activity不同的启动模式，对应不同的任务栈的存放；可通过 getTaskId()来获取任务栈的 ID，如果前面的任务栈已经清空，新开的任务栈ID+1，是自动增长的；首先来看下Task的定义，Google是这样定义Task的：Task实际上是一个Activity栈，通常用户感受的一个Application就是一个Task。从这个定义来看，Task跟Service或者其他Components是没有任何联系的，它只是针对Activity而言的。
- 同一程序不同的Activity是否可以放在不同的Task任务栈中？
    - 可以的。比如：启动模式里有个Singleinstance，可以运行在另外的单独的任务栈里面。用这个模式启动的activity，在内存中只有一份，这样就不会重复的开启。
    - 也可以在激活一个新的activity时候,给intent设置flag，Intent的flag添加FLAG_ACTIVITY_NEW_TASK，这个被激活的activity就会在新的task栈里面

## Activity启动流程

### 01.Launcher启动开启Activity
- 这个首先看LauncherActivity类中的onListItemClick方法
- Launcher启动之后会将各个应用包名和icon与app的name保存起来，然后执行icon的点击事件的时候调用startActivity方法：
    ```
    @Override
    protected void onListItemClick(ListView l, View v, int position, long id) {
        Intent intent = intentForPosition(position);
        startActivity(intent);
    }
    
    protected Intent intentForPosition(int position) {
        ActivityAdapter adapter = (ActivityAdapter) mAdapter;
        return adapter.intentForPosition(position);
    }
    
    public Intent intentForPosition(int position) {
        if (mActivitiesList == null) {
            return null;
        }
    
        Intent intent = new Intent(mIntent);
        ListItem item = mActivitiesList.get(position);
        intent.setClassName(item.packageName, item.className);
        if (item.extras != null) {
            intent.putExtras(item.extras);
        }
        return intent;
    }
    ```
    - 可以发现，我们在启动Activity的时候，执行的逻辑就是创建一个Intent对象，然后初始化Intent对象，使用隐式启动的方式启动该Acvitity，这里为什么不能使用显示启动的方式呢？
    - 这是因为Launcher程序启动的Activity一般都是启动一个新的应用进程，该进程与Launcher进程不是在同一个进程中，所以也就无法引用到启动的Activity字节码，自然也就无法启动该Activity了。
- 那么应用的图标是怎么和这个应用的Lancher Activity联系起来的呢？
    - 系统在启动的时候会启动PackageManagerService（包管理服务），所有的应用都是通过它安装的，PackageManagerService会对应用的AndroidManifest.xml进行解析，从而得到应用里所有的组件信息。Lanucher组件在启动过程中就会去向PackageManagerService查询包含以下信息的Activity组件。
    ```xml
    <intent-filter>
        <action android:text="android.intent.action.MAIN" />
        <category android:text="android.intent.category.LAUNCHER" />
    </intent-filter>
    ```
    - 并为每一个包含该信息的Activity组件创建一个快捷图标，由此两者便建立了联系。



### 02.查看startActivity方法
- 查看startActivity方法的具体实现：
    ``` java
    > MyActivity.startActivity()
    > Activity.startActivity()
    > Activity.startActivityForResult
    > Instrumentation.execStartActivty
    > ActivityManagerNative.getDefault().startActivityAsUser()
    ```


#### 2.1 Activity.startActivity()
- 在我们的Activity中调用startActivity方法，会执行Activity中的startActivity
    ``` java
    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
    ```
- 然后在Activity中的startActivity方法体里调用了startActivity的重载方法，这里我们看一下其重载方法的实现：
    ``` java
    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            startActivityForResult(intent, -1);
        }
    }
    ```
- 由于在上一步骤中我们传递的Bundle对象为空，所以这里我们执行的是else分支的逻辑，所以这里调用了startActivityForResult方法，并且传递的参数为intent和-1.
    - 注意：通过这里的代码我们可以发现，其实我们在Activity中调用startActivity的内部也是调用的startActivityForResult的。那么为什么调用startActivityForResult可以在Activity中回调onActivityResult而调用startActivity则不可以呢？可以发现其主要的区别是调用startActivity内部调用startActivityForResult传递的传输requestCode值为-1，也就是说我们在Activity调用startActivityForResult的时候传递的requestCode值为-1的话，那么onActivityResult是不起作用的。
    - 实际上，经测试requestCode的值小于0的时候都是不起作用的，所以当我们调用startActivityForResult的时候需要注意这一点。



#### 2.2 Activity.startActivityForResult
- 我们继续往下看，startActivityForResult方法的具体实现：
    ``` java
    public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }
    
            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
    ```
- 可以发现由于我们是第一次启动Activity，所以这里的mParent为空，所以会执行if分之，然后调用mInstrumentation.execStartActivity方法，并且这里需要注意的是，有一个判断逻辑：
    ```
    if (requestCode >= 0) {
    	mStartedActivity = true;
    }
    ```
    - 通过注释也验证了我们刚刚的说法即，调用startActivityForResult的时候只有requestCode的值大于等于0，onActivityResult才会被回调。


#### 2.3 Instrumentation.execStartActivity
- 然后我们看一下mInstrumentation.execStartActivity方法的实现。
    - 在查看execStartActivity方法之前，我们需要对mInstrumentation对象有一个了解？什么是Instrumentation？
    - Instrumentation是android系统中启动Activity的一个实际操作类，也就是说Activity在应用进程端的启动实际上就是Instrumentation执行的，那么为什么说是在应用进程端的启动呢？
    - 实际上activity的启动分为应用进程端的启动和SystemServer服务进程端的启动的，多个应用进程相互配合最终完成了Activity在系统中的启动的，而在应用进程端的启动实际的操作类就是Instrumentation来执行的，可能还是有点绕口，没关系，随着我们慢慢的解析大家就会对Instrumentation的认识逐渐加深的。
- 可以发现execStartActivity方法传递的几个参数：
    ```
    this，为启动Activity的对象；
    contextThread，为Binder对象，是主进程的context对象；
    token，也是一个Binder对象，指向了服务端一个ActivityRecord对象；
    target，为启动的Activity；
    intent，启动的Intent对象；
    requestCode，请求码；
    options，参数；
    ```
- 这样就调用了Instrument.execStartActivity方法了：
    ```
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
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
- 我们发现在这个方法中主要调用ActivityManagerNative.getDefault().startActivity方法，那么ActivityManagerNative又是个什么鬼呢？查看一下getDefault()对象的实现：
    ```
    static public IActivityManager getDefault() {
        return gDefault.get();
    }
    ```
- 好吧，相当之简单直接返回的是gDefault.get()，那么gDefault又是什么呢？
    ```
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
    ```
- 可以发现启动过asInterface()方法创建，然后我们继续看一下asInterface方法的实现：
    ```
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        return new ActivityManagerProxy(obj);
    }
    ```
- 最后直接返回一个ActivityManagerProxy对象，而ActivityManagerProxy继承与IActivityManager，到了这里就引出了我们android系统中很重要的一个概念：Binder机制。



#### 2.4 IActivityManager说明
- 我们知道应用进程与SystemServer进程属于两个不同的进程，进程之间需要通讯，android系统采取了自身设计的Binder机制，这里的ActivityManagerProxy和ActivityManagerNative都是继承与IActivityManager的而SystemServer进程中的ActivityManagerService对象则继承与ActivityManagerNative。
- 简单的表示：Binder接口 --> ActivityManagerNative/ActivityManagerProxy --> ActivityManagerService；
- 这样，ActivityManagerNative与ActivityManagerProxy相当于一个Binder的客户端而ActivityManagerService相当于Binder的服务端，这样当ActivityManagerNative调用接口方法的时候底层通过Binder driver就会将请求数据与请求传递给server端，并在server端执行具体的接口逻辑。需要注意的是Binder机制是单向的，是异步的，也就是说只能通过client端向server端传递数据与请求而不同等待服务端的返回，也无法返回，那如果SystemServer进程想向应用进程传递数据怎么办？这时候就需要重新定义一个Binder请求以SystemServer为client端，以应用进程为server端，这样就是实现了两个进程之间的双向通讯。


#### 2.5 ActivityManagerService
- 说了这么多我们知道这里的ActivityManagerNative是ActivityManagerService在应用进程的一个client就好了，通过它就可以调用ActivityManagerService的方法了。
- 继续往下卡，我们调用的是：
    ```
    int result = ActivityManagerNative.getDefault()
        .startActivity(whoThread, who.getBasePackageName(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()),
                token, target != null ? target.mEmbeddedID : null,
                requestCode, 0, null, options);
    ```
- 这里通过我们刚刚的分析，ActivityManagerNative.getDefault()方法会返回一个ActivityManagerProxy对象，那么我们看一下ActivityManagerProxy对象的startActivity方法：
    ```
    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(callingPackage);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
    ```
- 这里就涉及到了具体的Binder数据传输机制了，我们不做过多的分析，知道通过数据传输之后就会调用SystemServer进程的ActivityManagerService的startActivity就好了。



### 03.ActivityManagerService详谈
- ActivityManagerService接收启动Activity的请求
    ``` java
    > ActivityManagerService.startActivity() 
    > ActvityiManagerService.startActivityAsUser() 
    > ActivityStackSupervisor.startActivityMayWait() 
    > ActivityStackSupervisor.startActivityLocked() 
    > ActivityStackSupervisor.startActivityUncheckedLocked() 
    > ActivityStackSupervisor.startActivityLocked() 
    > ActivityStackSupervisor.resumeTopActivitiesLocked()
    > ActivityStackSupervisor.resumeTopActivityInnerLocked() 
    ```

#### 3.1 ActivityManagerService.startActivity() 
- 首先看一下ActivityManagerService.startActivity的具体实现；
    ``` java
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }
    ```
- 可以看到，该方法并没有实现什么逻辑，直接调用了startActivityAsUser方法，我们继续看一下startActivityAsUser方法的实现：
    ``` java
    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
    ```
- 可以看到这里只是进行了一些关于userid的逻辑判断，然后就调用mStackSupervisor.startActivityMayWait方法，下面我们来看一下这个方法的具体实现：
    ``` java
    final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {    
            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);
            return res;
    }
    ```
- 这个方法中执行了启动Activity的一些其他逻辑判断，在经过判断逻辑之后调用startActivityLocked方法：
    ``` java
    final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;
        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);
        return err;
    }
    ```
- 这个方法中主要构造了ActivityManagerService端的Activity对象-->ActivityRecord，并根据Activity的启动模式执行了相关逻辑。然后调用了startActivityUncheckedLocked方法：
    ``` java
    final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {
        ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
        targetStack.mLastPausedActivity = null;
        targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
        if (!launchTaskBehind) {
            // Don't set focus on an activity that's going to the back.
            mService.setFocusedActivityLocked(r, "startedActivity");
        }
        return ActivityManager.START_SUCCESS;
    }
    ```
- startActivityUncheckedLocked方法中只要执行了不同启动模式不同栈的处理，并最后调用了startActivityLocked的重载方法：
    ``` java
    final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
    	...
        if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
    ```
- 这个startActivityLocked方法主要执行初始化了windowManager服务，然后调用resumeTopActivitiesLocked方法：
    ``` java
    boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }
    
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (stack == targetStack) {
                    continue;
                }
                if (isFrontStack(stack)) {
                    stack.resumeTopActivityLocked(null);
                }
            }
        }
        return result;
    }
    ```
- 可以发现经过循环逻辑判断之后，最终调用了resumeTopActivityLocked方法：
    ``` java
    final boolean resumeTopActivityLocked(ActivityRecord prev) {
        return resumeTopActivityLocked(prev, null);
    }
    ```
- 然后调用：
    ``` java
    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }
    
        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
                mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
                mService.updateSleepIfNeededLocked();
            }
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
    ```
- 继续调用resumeTopActivityInnerLocked方法：
    ``` java
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
    	if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }        ...
        return true;
    }
    ```


### 04.执行栈顶Activity的onPause方法
- 经过一系列处理逻辑之后最终调用了startPausingLocked方法，这个方法作用就是让系统中栈中的Activity执行onPause方法。
    ``` java
    >ActivityStack.startPausingLocked()
    >IApplicationThread.schudulePauseActivity()
    >ActivityThread.sendMessage()
    >ActivityThread.H.sendMessage();
    >ActivityThread.H.handleMessage()
    >ActivityThread.handlePauseActivity()
    >ActivityThread.performPauseActivity()
    >Activity.performPause()
    >Activity.onPause()
    >ActivityManagerNative.getDefault().activityPaused(token)
    >ActivityManagerService.activityPaused()
    >ActivityStack.activityPausedLocked()
    >ActivityStack.completePauseLocked()
    >ActivityStack.resumeTopActivitiesLocked()
    >ActivityStack.resumeTopActivityLocked()
    >ActivityStack.resumeTopActivityInnerLocked()
    >ActivityStack.startSpecificActivityLocked
    ```
- 好吧，方法比较多也比较乱，首先来看startPausingLocked方法：
    ``` java
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
        ...
        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                        prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName);
                mService.updateUsageStats(prev, false);
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }
        ...
    }
    ```
    - 可以看到这里执行了pre.app.thread.schedulePauseActivity方法，通过分析不难发现这里的thread是一个IApplicationThread类型的对象，而在ActivityThread中也定义了一个ApplicationThread的类，其继承了IApplicationThread，并且都是Binder对象，不难看出这里的IAppcation是一个Binder的client端而ActivityThread中的ApplicationThread是一个Binder对象的server端，所以通过这里的thread.schedulePauseActivity实际上调用的就是ApplicationThread的schedulePauseActivity方法。
    - 这里的ApplicationThread可以和ActivityManagerNative对于一下：
    - 通过ActivityManagerNative-->ActivityManagerService实现了应用进程与SystemServer进程的通讯。通过AppicationThread——IApplicationThread实现了SystemServer进程与应用进程的通讯。
- 然后我们继续看一下ActivityThread中schedulePauseActivity的具体实现：
    ``` java
    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
        sendMessage(
                finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                token,
                (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                configChanges);
    }
    ```
- 发送了PAUSE_ACTIVITY_FINISHING消息，然后看一下sendMessage的实现方法：
    ``` java
    private void sendMessage(int what, Object obj, int arg1, int arg2) {
        sendMessage(what, obj, arg1, arg2, false);
    }
    ```
- 调用了其重载方法：
    ``` java
    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
            + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
    ```
- 最终调用了mH的sendMessage方法，mH是在ActivityThread中定义的一个Handler对象，主要处理SystemServer进程的消息，我们看一下其handleMessge方法的实现：
    ```
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            ...
            case PAUSE_ACTIVITY_FINISHING:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                handlePauseActivity((IBinder)msg.obj, true, (msg.arg1&1) != 0, msg.arg2,
                        (msg.arg1&1) != 0);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
    		...
    }
    ```
- 可以发现其调用了handlePauseActivity方法：
    ```
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
- 然后在方法体内部通过调用performPauseActivity方法来实现对栈顶Activity的onPause生命周期方法的回调，可以具体看一下他的实现：
    ```
    final Bundle performPauseActivity(IBinder token, boolean finished,
            boolean saveState) {
        ActivityClientRecord r = mActivities.get(token);
        return r != null ? performPauseActivity(r, finished, saveState) : null;
    }
    ```
- 然后调用其重载方法：
    ```
    final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState) {
        ...
        mInstrumentation.callActivityOnPause(r.activity);
        ...
    
        return !r.activity.mFinished && saveState ? r.state : null;
    }
    ```
- 这样回到了mInstrumentation的callActivityOnPuase方法：
    ```
    public void callActivityOnPause(Activity activity) {
        activity.performPause();
    }
    ```
- 原来最终回调到了Activity的performPause方法：
    ```
    final void performPause() {
        mDoReportFullyDrawn = false;
        mFragments.dispatchPause();
        mCalled = false;
        onPause();
        mResumed = false;
        if (!mCalled && getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.GINGERBREAD) {
            throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onPause()");
        }
        mResumed = false;
    }
    ```
    - 终于，回调到了Activity的onPause方法，Activity生命周期中的第一个生命周期方法终于被我们找到了。。。。也就是说我们在启动一个Activity的时候最先被执行的是栈顶的Activity的onPause方法。记住这点吧，面试的时候经常会问到类似的问题。
- 然后回到我们的handlePauseActivity方法
    - 在该方法的最后面执行了ActivityManagerNative.getDefault().activityPaused(token);方法，这是应用进程告诉服务进程，栈顶Activity已经执行完成onPause方法了，通过前面我们的分析，我们知道这句话最终会被ActivityManagerService的activityPaused方法执行。
    ```
    @Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
    ```
- 可以发现，该方法内部会调用ActivityStack的activityPausedLocked方法，好吧，继续看一下activityPausedLocked方法的实现：
    ```
    final void activityPausedLocked(IBinder token, boolean timeout) {
            if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSED: " + r
                    + (timeout ? " (due to timeout)" : " (pause complete)"));
            completePauseLocked(true);
    }
    ```
- 然后执行了completePauseLocked方法：
    ```
    private void completePauseLocked(boolean resumeNext) {    
        if (resumeNext) {
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!mService.isSleepingOrShuttingDown()) {
                mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
            } else {
                mStackSupervisor.checkReadyForSleepLocked();
                ActivityRecord top = topStack.topRunningActivityLocked(null);
                if (top == null || (prev != null && top != prev)) {
                    mStackSupervisor.resumeTopActivitiesLocked(topStack, null, null);
                }
            }
        }
    }
    ```
- 经过了一系列的逻辑之后，又调用了resumeTopActivitiesLocked方法，又回到了第二步中解析的方法中了，这样经过
    ```
    resumeTopActivitiesLocked --> 
    ActivityStack.resumeTopActivityLocked() --> 
    resumeTopActivityInnerLocked --> 
    startSpecificActivityLocked
    ```
- 我们看一下startSpecificActivityLocked的具体实现：
    ```
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        ProcessRecord app = mService.getProcessRecordLocked(r.processName, r.info.applicationInfo.uid, true);
        r.task.stack.setLaunchTime(r);
        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
    
            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
    
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
    ```
- 可以发现在这个方法中，首先会判断一下需要启动的Activity所需要的应用进程是否已经启动，若启动的话，则直接调用realStartAtivityLocked方法，否则调用startProcessLocked方法，用于启动应用进程。
    - 这样关于启动Activity时的第三步骤就已经执行完成了，这里主要是实现了对栈顶Activity执行onPause方法，而这个方法首先判断需要启动的Activity所属的进程是否已经启动，若已经启动则直接调用启动Activity的方法，否则将先启动Activity的应用进程，然后在启动该Activity。



### 06.执行启动Activity重点逻辑
- 大概流程如下所示
    >ActivityStackSupervisor.attachApplicationLocked()
    >ActivityStackSupervisor.realStartActivityLocked()
    >IApplicationThread.scheduleLauncherActivity()
    >ActivityThread.sendMessage()
    >ActivityThread.H.sendMessage()
    >ActivityThread.H.handleMessage()
    >ActivityThread.handleLauncherActivity()
    >ActivityThread.performLauncherActivity()
    >Instrumentation.callActivityOnCreate()
    >Activity.onCreate()
    >ActivityThread.handleResumeActivity()
    >ActivityThread.performResumeActivity()
    >Activity.performResume()
    >Instrumentation.callActivityOnResume()
    >Activity.onResume()
    >ActivityManagerNative.getDefault().activityResumed(token)
- 首先看一下attachApplicationLocked方法的实现：
    ```
    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFrontStack(stack)) {
                    continue;
                }
                ActivityRecord hr = stack.topRunningActivityLocked(null);
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Exception in new application when starting activity "
                                  + hr.intent.getComponent().flattenToShortString(), e);
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisibleLocked(null, 0);
        }
        return didSomething;
    }
    ```
- 可以发现其内部调用了realStartActivityLocked方法，通过名字可以知道这个方法应该就是用来启动Activity的，看一下这个方法的实现逻辑：
    ```
    final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {
    
            app.forceProcessStateUpTo(mService.mTopProcessState);
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
        return true;
    }
    ```
- 可以发现与第三步执行栈顶Activity onPause时类似，这里也是通过调用IApplicationThread的方法实现的，这里调用的是scheduleLauncherActivity方法，所以真正执行的是ActivityThread中的scheduleLauncherActivity，所以我们看一下ActivityThread中的scheduleLauncherActivity的实现：
    ```
    @Override
    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
    
        updateProcessState(procState, false);
    
        ActivityClientRecord r = new ActivityClientRecord();
    
        r.token = token;
        r.ident = ident;
        r.intent = intent;
        r.referrer = referrer;
        r.voiceInteractor = voiceInteractor;
        r.activityInfo = info;
        r.compatInfo = compatInfo;
        r.state = state;
        r.persistentState = persistentState;
    
        r.pendingResults = pendingResults;
        r.pendingIntents = pendingNewIntents;
    
        r.startsNotResumed = notResumed;
        r.isForward = isForward;
    
        r.profilerInfo = profilerInfo;
    
        r.overrideConfig = overrideConfig;
        updatePendingConfiguration(curConfig);
    
        sendMessage(H.LAUNCH_ACTIVITY, r);
    }
    ```
- 还是那套逻辑，ActivityThread接收到SystemServer进程的消息之后会通过其内部的Handler对象分发消息，经过一系列的分发之后调用了ActivityThread的handleLaunchActivity方法：
    ```
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    
        Activity a = performLaunchActivity(r, customIntent);
    
        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);
        }
    }
    ```
- 可以发现这里调用了performLauncherActivity，看名字应该就是执行Activity的启动操作了。。。
    ```
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {    
    Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }    
        activity.mCalled = false;
        if (r.isPersistable()) {
           mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
           mInstrumentation.callActivityOnCreate(activity, r.state);
        }
    	if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
        return activity;
    }
    ```
- 可以发现这里我们需要的Activity对象终于是创建出来了，而且他是以反射的机制创建的，现在还不太清楚为啥google要以反射的方式创建Activity，先不看这些，然后在代码中其调用Instrumentation的callActivityOnCreate方法。
    ```
    public void callActivityOnCreate(Activity activity, Bundle icicle,
            PersistableBundle persistentState) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
    ```
- 然后执行activity的performCreate方法。。。
    ```
    final void performCreate(Bundle icicle) {
        onCreate(icicle);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }
    ```
- 在回到我们的performLaunchActivity方法，其在调用了mInstrumentation.callActivityOnCreate方法之后又调用了activity.performStart();方法，好吧，看一下他的实现方式：
    ```
    final void performStart() {
        mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
        mFragments.noteStateNotSaved();
        mCalled = false;
        mFragments.execPendingActions();
        mInstrumentation.callActivityOnStart(this);
        if (!mCalled) {
            throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onStart()");
        }
        mFragments.dispatchStart();
        mFragments.reportLoaderStart();
        mActivityTransitionState.enterReady(this);
    }
    ```
- 还是通过Instrumentation调用callActivityOnStart方法：
    ```
    public void callActivityOnStart(Activity activity) {
        activity.onStart();
    }
    ```
- 还是回到我们刚刚的handleLaunchActivity方法，在调用完performLaunchActivity方法之后，其有吊用了handleResumeActivity方法，好吧，看名字应该是回调Activity的onResume方法的。
    ```
    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;
    
        // TODO Push resumeArgs into the activity for consideration
        ActivityClientRecord r = performResumeActivity(token, clearHide);
    
        if (r != null) {
            final Activity a = r.activity;
    
            if (localLOGV) Slog.v(
                TAG, "Resume " + r + " started activity: " +
                a.mStartedActivity + ", hideForNow: " + r.hideForNow
                + ", finished: " + a.mFinished);
    
            final int forwardBit = isForward ?
                    WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
    
            // If the window hasn't yet been added to the window manager,
            // and this guy didn't finish itself or start another activity,
            // then go ahead and add the window.
            boolean willBeVisible = !a.mStartedActivity;
            if (!willBeVisible) {
                try {
                    willBeVisible = ActivityManagerNative.getDefault().willActivityBeVisible(
                            a.getActivityToken());
                } catch (RemoteException e) {
                }
            }
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }
    
            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }
    
            // Get rid of anything left hanging around.
            cleanUpPendingRemoveWindows(r);
    
            // The window is now visible if it has been added, we are not
            // simply finishing, and we are not starting another activity.
            if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                if (r.newConfig != null) {
                    r.tmpConfig.setTo(r.newConfig);
                    if (r.overrideConfig != null) {
                        r.tmpConfig.updateFrom(r.overrideConfig);
                    }
                    if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity "
                            + r.activityInfo.name + " with newConfig " + r.tmpConfig);
                    performConfigurationChanged(r.activity, r.tmpConfig);
                    freeTextLayoutCachesIfNeeded(r.activity.mCurrentConfig.diff(r.tmpConfig));
                    r.newConfig = null;
                }
                if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward="
                        + isForward);
                WindowManager.LayoutParams l = r.window.getAttributes();
                if ((l.softInputMode
                        & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                        != forwardBit) {
                    l.softInputMode = (l.softInputMode
                            & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                            | forwardBit;
                    if (r.activity.mVisibleFromClient) {
                        ViewManager wm = a.getWindowManager();
                        View decor = r.window.getDecorView();
                        wm.updateViewLayout(decor, l);
                    }
                }
                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                    r.activity.makeVisible();
                }
            }
    
            if (!r.onlyLocalRequest) {
                r.nextIdle = mNewActivities;
                mNewActivities = r;
                if (localLOGV) Slog.v(
                    TAG, "Scheduling idle handler for " + r);
                Looper.myQueue().addIdleHandler(new Idler());
            }
            r.onlyLocalRequest = false;
    
            // Tell the activity manager we have resumed.
            if (reallyResume) {
                try {
                    ActivityManagerNative.getDefault().activityResumed(token);
                } catch (RemoteException ex) {
                }
            }
    
        } else {
            // If an exception was thrown when trying to resume, then
            // just end this activity.
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {
            }
        }
    }
    ```
- 可以发现其resumeActivity的逻辑调用到了performResumeActivity方法，我们来看一下performResumeActivity是如何实现的。
    ```
    public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide) {
        ActivityClientRecord r = mActivities.get(token);
        if (localLOGV) Slog.v(TAG, "Performing resume of " + r
                + " finished=" + r.activity.mFinished);
        if (r != null && !r.activity.mFinished) {
            if (clearHide) {
                r.hideForNow = false;
                r.activity.mStartedActivity = false;
            }
            try {
                r.activity.onStateNotSaved();
                r.activity.mFragments.noteStateNotSaved();
                if (r.pendingIntents != null) {
                    deliverNewIntents(r, r.pendingIntents);
                    r.pendingIntents = null;
                }
                if (r.pendingResults != null) {
                    deliverResults(r, r.pendingResults);
                    r.pendingResults = null;
                }
                r.activity.performResume();
    
                EventLog.writeEvent(LOG_AM_ON_RESUME_CALLED,
                        UserHandle.myUserId(), r.activity.getComponentName().getClassName());
    
                r.paused = false;
                r.stopped = false;
                r.state = null;
                r.persistentState = null;
            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException(
                        "Unable to resume activity "
                        + r.intent.getComponent().toShortString()
                        + ": " + e.toString(), e);
                }
            }
        }
        return r;
    }
    ```
- 在方法体中，最终调用了r.activity.performResume();方法，好吧，这个方法是Activity中定义的方法，我们需要在Activity中查看这个方法的具体实现：
    ```
    final void performResume() {
        ...
        mInstrumentation.callActivityOnResume(this);
        ...
    }
    ```
- 又是熟悉的味道，通过Instrumentation来调用了callActivityOnResume方法。。。
    ```
    public void callActivityOnResume(Activity activity) {
        activity.mResumed = true;
        activity.onResume();
        
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    am.match(activity, activity, activity.getIntent());
                }
            }
        }
    }
    ```




### 07.启动流程大概总结一下
- 大概如下所示
    - Activity的启动流程一般是通过调用startActivity或者是startActivityForResult来开始的
    - startActivity内部也是通过调用startActivityForResult来启动Activity，只不过传递的requestCode小于0
    - Activity的启动流程涉及到多个进程之间的通讯这里主要是ActivityThread与ActivityManagerService之间的通讯
    - ActivityThread向ActivityManagerService传递进程间消息通过ActivityManagerNative，ActivityManagerService向ActivityThread进程间传递消息通过IApplicationThread。
    - ActivityManagerService接收到应用进程创建Activity的请求之后会执行初始化操作，解析启动模式，保存请求信息等一系列操作。
    - ActivityManagerService保存完请求信息之后会将当前系统栈顶的Activity执行onPause操作，并且IApplication进程间通讯告诉应用程序继承执行当前栈顶的Activity的onPause方法；
    - ActivityThread接收到SystemServer的消息之后会统一交个自身定义的Handler对象处理分发；
    - ActivityThread执行完栈顶的Activity的onPause方法之后会通过ActivityManagerNative执行进程间通讯告诉ActivityManagerService，栈顶Activity已经执行完成onPause方法，继续执行后续操作；
    - ActivityManagerService会继续执行启动Activity的逻辑，这时候会判断需要启动的Activity所属的应用进程是否已经启动，若没有启动则首先会启动这个Activity的应用程序进程；
    - ActivityManagerService会通过socket与Zygote继承通讯，并告知Zygote进程fork出一个新的应用程序进程，然后执行ActivityThread的mani方法；
    - 在ActivityThread.main方法中执行初始化操作，初始化主线程异步消息，然后通知ActivityManagerService执行进程初始化操作；
    - ActivityManagerService会在执行初始化操作的同时检测当前进程是否有需要创建的Activity对象，若有的话，则执行创建操作；
    - ActivityManagerService将执行创建Activity的通知告知ActivityThread，然后通过反射机制创建出Activity对象，并执行Activity的onCreate方法，onStart方法，onResume方法；
    - ActivityThread执行完成onResume方法之后告知ActivityManagerService onResume执行完成，开始执行栈顶Activity的onStop方法；
    - ActivityManagerService开始执行栈顶的onStop方法并告知ActivityThread；
    - ActivityThread执行真正的onStop方法；

## Activity布局创建


### 01.前沿介绍
- 大家都知道在Android体系中Activity扮演了一个界面展示的角色
    - 这也是它与android中另外一个很重要的组件Service最大的不同，但是这个展示的界面的功能是Activity直接控制的么？
    - 界面的布局文件是如何加载到内存并被Activity管理的？android中的View是一个怎样的概念？加载到内存中的布局文件是如何绘制出来的？
- 其实Activity对界面布局的管理是都是通过Window对象来实现的
    - Window对象，顾名思义就是一个窗口对象，而Activity从用户角度就是一个个的窗口实例，因此不难想象每个Activity中都对应着一个Window对象，而这个Window对象就是负责加载显示界面的。
    - 至于window对象是如何展示不同的界面的，那是通过定义不同的View组件实现不同的界面展示。 

### 02.handleLaunchActivity
- 当ActivityManagerService接收到启动Activity的请求之后会通过IApplicationThread进程间通讯告知ApplicationThread并执行handleLauncherActivity方法，这里可以看一下其具体实现：
    - 可以发现这里的handleLauncherActivity方法内部调用了performLaunchActivity方法。
    ```
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;
    
        if (r.profilerInfo != null) {
            mProfiler.setProfiler(r.profilerInfo);
            mProfiler.startProfiling();
        }
    
        // Make sure we are running with the most recent config.
        handleConfigurationChanged(null, null);
    
        if (localLOGV) Slog.v(
            TAG, "Handling launch of " + r);
    
        // Initialize before creating the activity
        WindowManagerGlobal.initialize();
    
        Activity a = performLaunchActivity(r, customIntent);
    
        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);
    
            if (!r.activity.mFinished && r.startsNotResumed) {
                // The activity manager actually wants this one to start out
                // paused, because it needs to be visible but isn't in the
                // foreground.  We accomplish this by going through the
                // normal startup (because activities expect to go through
                // onResume() the first time they run, before their window
                // is displayed), and then pausing it.  However, in this case
                // we do -not- need to do the full pause cycle (of freezing
                // and such) because the activity manager assumes it can just
                // retain the current state it has.
                try {
                    r.activity.mCalled = false;
                    mInstrumentation.callActivityOnPause(r.activity);
                    // We need to keep around the original state, in case
                    // we need to be created again.  But we only do this
                    // for pre-Honeycomb apps, which always save their state
                    // when pausing, so we can not have them save their state
                    // when restarting from a paused state.  For HC and later,
                    // we want to (and can) let the state be saved as the normal
                    // part of stopping the activity.
                    if (r.isPreHoneycomb()) {
                        r.state = oldState;
                    }
                    if (!r.activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPause()");
                    }
    
                } catch (SuperNotCalledException e) {
                    throw e;
    
                } catch (Exception e) {
                    if (!mInstrumentation.onException(r.activity, e)) {
                        throw new RuntimeException(
                                "Unable to pause activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
                    }
                }
                r.paused = true;
            }
        } else {
            // If there was an error, for any reason, tell the activity
            // manager to stop us.
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {
                // Ignore
            }
        }
    }
    ```

### 03.performLaunchActivity
- 这个方法也是具体启动Activity的方法，我们来看一下它的具体实现逻辑：
    - 从代码中可以看到这里是通过反射的机制创建的Activity，并调用了Activity的attach方法，那么这里的attach方法是做什么的呢？
    ```
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }
        
    	...
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    
            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());
    
            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);
    
        ...
    
        return activity;
    }
    ```

### 04.activity.attach
- 我们继续来看一下attach方法的实现逻辑：
    ```
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
        attachBaseContext(context);
    
        mFragments.attachHost(null /*parent*/);
    
        mWindow = new PhoneWindow(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();
    
        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }
    
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
    ```
- 可以看到在attach方法这里初始化了一些Activity的成员变量，主要是mWindow对象，并且mWindow的成员实例是PhoneWindow实例，这样也从侧面说明了一个Activity对应着一个Window对象。除了window对象还初始化了一些Activity的其他成员变量，这里不再做讨论，继续回到我们的performLaunchActivity方法，在调用了Activity的attach方法之后又调用了：
    ```
    mInstrumentation.callActivityOnCreate(activity, r.state);
    ```
- 这里的mInstrumentation是类Instrumentation，每个应用进程对应着一个Instrumentation和一个ActivityThread，Instrumentation就是具体操作Activity回调其生命周期方法的，我们这里看一下它的callActivityOnCreate方法的实现：
    ```
    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
    ```
- 这里代码比较简洁，preOerformCreate方法和postPerformCreate方法我们这里暂时不管，主要的执行逻辑是调用了activity.performCreate方法，我们来看一下Activity的performCreate方法的实现：
    ```
    final void performCreate(Bundle icicle) {
        onCreate(icicle);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }
    ```
- 原来onCreate的生命周期方法是在这里回调的，其实这里的逻辑在前面几篇文章中有讲述，也可以参考前面的文章。



### 05.Activity的onCreate方法
- 至此就回调到了我们Activity的onCreate方法，大家平时在重写onCreate方法的时候，怎么加载布局文件的呢？这里看一下我们的onCreate方法的典型写法：
    ```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
    ```
- 无论我们怎么变化，我们的onCreate方法一般都是会调用这两句话的吧？那么这里的两段代码分辨是什么含义呢？我们首先看一下super.onCreate方法的实现逻辑，由于我们的Activity类继承与Activity，所以这里的super.onCreate方法，就是调用的Activity.onCreate方法，好吧，既然这样我们来看一下Activity的onCreate方法：
    ```
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        if (DEBUG_LIFECYCLE) Slog.v(TAG, "onCreate " + this + ": " + savedInstanceState);
        if (mLastNonConfigurationInstances != null) {
            mFragments.restoreLoaderNonConfig(mLastNonConfigurationInstances.loaders);
        }
        if (mActivityInfo.parentActivityName != null) {
            if (mActionBar == null) {
                mEnableDefaultActionBarUp = true;
            } else {
                mActionBar.setDefaultDisplayHomeAsUpEnabled(true);
            }
        }
        if (savedInstanceState != null) {
            Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
            mFragments.restoreAllState(p, mLastNonConfigurationInstances != null
                    ? mLastNonConfigurationInstances.fragments : null);
        }
        mFragments.dispatchCreate();
        getApplication().dispatchActivityCreated(this, savedInstanceState);
        if (mVoiceInteractor != null) {
            mVoiceInteractor.attachActivity(this);
        }
        mCalled = true;
    }
    ```
- 可以发现，Activity的onCreate方法主要是做了一些Acitivty的初始化操作，那么如果我们不在自己的Activity调用super.onCreate方法呢？好吧，尝试之后，AndroidStudio在打开的Acitivty的onCreate方法中如果不调用super.onCreate方法的话，会报错。。。
    ```
    FATAL EXCEPTION: main                                                              Process: com.example.aaron.helloworld, PID: 18001                                                                 android.util.SuperNotCalledException: Activity {com.example.aaron.helloworld/com.example.aaron.helloworld.SecondActivity} did not call through to super.onCreate()                                                                    at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2422)                                                                            at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2528)                                                                               at android.app.ActivityThread.access$800(ActivityThread.java:169)                                                                              at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1421)                                                                             at android.os.Handler.dispatchMessage(Handler.java:111)                                                                            at android.os.Looper.loop(Looper.java:194)                                                                           at android.app.ActivityThread.main(ActivityThread.java:5552)                                                                        at java.lang.reflect.Method.invoke(Native Method)                                                                        at java.lang.reflect.Method.invoke(Method.java:372)                                                                      at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:964)                                                                       at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:759)
    ```
- 可以看到如果不调用super.onCreate方法的话，会在Activity的performLaunchActivity中报错，我们知道这里的performLaunchActivity方法就是我们启动Activity的时候回回调的方法，我们找找方法体实现中throws的Exception。。。
    ```
    activity.mCalled = false;
    if (r.isPersistable()) {
        mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
    } else {
        mInstrumentation.callActivityOnCreate(activity, r.state);
    }
    if (!activity.mCalled) {
        throw new SuperNotCalledException(
            "Activity " + r.intent.getComponent().toShortString() +
            " did not call through to super.onCreate()");
    }
    ```
- 在Activity的performLaunchActivity方法中，我们在调用了Activity的onCreate方法之后会执行一个判断逻辑，若Activity的mCalled为false，则会抛出我们刚刚捕获的异常，那么这个mCalled成员变量是在什么时候被赋值的呢？好吧，就是在Activity的onCreate方法赋值的，所以我们在实现自己的Activity的时候只有调用了super.onCreate方法才不会抛出这个异常，反过来说，我们实现自己的Actiivty，那么一定要在onCreate方法中调用super.onCreate方法。


### 06.setContentView
- 然后我们在看一下onCreate中的setContentView方法，这里的参数就是一个Layout布局文件，可以发现这里的setContentView方法就是Acitivty中的setContentView，好吧我们来看一下Activity中setContentView的实现：
    ```
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
    ```
- 这里的getWindow方法就是获取Acitivty的mWindow成员变量，从刚刚我们在Activity.attach方法我们知道这里的mWindow的实例是PhoneWindow，所以这里调用的其实是PhoneWindow的setConentView方法，然后我们看一下PhoneWindow的setContentView是如何实现的。
    ```
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }
    
        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
    ```
- 这里的mContentParent对象是一个View对象，由于第一次mContentParent为空，所以执行installerDector方法，这里我们看一下installerDector方法的具体实现：
    ```
    private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        }
        ...
    }
    ```
- 这里的mDector是一个DectorView对象，而DectorView继承与FrameLayout，所以这里的mDector其实就是一个FrameLayout对象，并通过调用generateDector()方法初始化，我们继续看一下generateDector方法的具体实现：
    ```
    protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }
    ```
- 就是通过new的方式创建了一个DectorView对象，然后我们继续看installDector方法：
    ```
    if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
    ```
- 这里初始化了mContentParent对象，这是一个View对象，我们调用了generateLayout方法，好吧，来看一下generateLayout方法的具体实现：
    ```
    protected ViewGroup generateLayout(DecorView decor) {
        ...
        // Inflate the window decor.
    
        int layoutResource;
        int features = getLocalFeatures();
        // System.out.println("Features: 0x" + Integer.toHexString(features));
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss;
        } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_title_icons;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
            // System.out.println("Title Icons!");
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            // Special case for a window with only a progress bar (and title).
            // XXX Need to have a no-title version of embedded windows.
            layoutResource = R.layout.screen_progress;
            // System.out.println("Progress!");
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            // Special case for a window with a custom title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_custom_title;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            // If no other features and not embedded, only need a title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
                layoutResource = a.getResourceId(
                        R.styleable.Window_windowActionBarFullscreenDecorLayout,
                        R.layout.screen_action_bar);
            } else {
                layoutResource = R.layout.screen_title;
            }
            // System.out.println("Title!");
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            // Embedded, so no decoration is needed.
            layoutResource = R.layout.screen_simple;
            // System.out.println("Simple!");
        }
    
        mDecor.startChanging();
    
        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in;
    
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }
    
        if ((features & (1 << FEATURE_INDETERMINATE_PROGRESS)) != 0) {
            ProgressBar progress = getCircularProgressBar(false);
            if (progress != null) {
                progress.setIndeterminate(true);
            }
        }
    
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            registerSwipeCallbacks();
        }
    
        // Remaining setup -- of background and title -- that only applies
        // to top-level windows.
        if (getContainer() == null) {
            final Drawable background;
            if (mBackgroundResource != 0) {
                background = getContext().getDrawable(mBackgroundResource);
            } else {
                background = mBackgroundDrawable;
            }
            mDecor.setWindowBackground(background);
    
            final Drawable frame;
            if (mFrameResource != 0) {
                frame = getContext().getDrawable(mFrameResource);
            } else {
                frame = null;
            }
            mDecor.setWindowFrame(frame);
    
            mDecor.setElevation(mElevation);
            mDecor.setClipToOutline(mClipToOutline);
    
            if (mTitle != null) {
                setTitle(mTitle);
            }
    
            if (mTitleColor == 0) {
                mTitleColor = mTextColor;
            }
            setTitleColor(mTitleColor);
        }
    
        mDecor.finishChanging();
    
        return contentParent;
    }
    ```
- 可以发现这里就是通过调用LayoutInflater.inflate方法来加载布局文件到内存中，关于LayoutInflater.inflater是如何加载布局文件的，并且，通过对代码的分析，我们发现PhoneWindow中的几个成员变量：mDector，mContentRoot，mContentParent的关系
    - mDector --> mContentRoot --> mContentParent（包含）
    - 并且我们来看一下典型的布局文件：
    ```
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:fitsSystemWindows="true"
        android:orientation="vertical">
        <ViewStub android:id="@+id/action_mode_bar_stub"
                  android:inflatedId="@+id/action_mode_bar"
                  android:layout="@layout/action_mode_bar"
                  android:layout_width="match_parent"
                  android:layout_height="wrap_content"
                  android:theme="?attr/actionBarTheme" />
        <FrameLayout
             android:id="@android:id/content"
             android:layout_width="match_parent"
             android:layout_height="match_parent"
             android:foregroundInsidePadding="false"
             android:foregroundGravity="fill_horizontal|top"
             android:foreground="?android:attr/windowContentOverlay" />
    </LinearLayout>
    ```
    - 这里就是整个Activity加载的跟布局文件：screen_simple.xml，其中ViewStub对应着Activity中的titleBar而这里的FrameLayout里面主要用于填充内容。
- 然后我们具体看一下LayoutInflater.inflate方法：
    ```
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }
    ```
- 这里调用了inflate的重载方法。。。
    ```
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
    
            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;
    
            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
    
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }
    
                final String name = parser.getName();
                
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }
    
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
    
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
    
                    ViewGroup.LayoutParams params = null;
    
                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }
    
                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }
    
                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);
    
                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }
    
                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }
    
                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }
    
            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (Exception e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                                + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }
    
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    
            return result;
        }
    }
    ```
- 通过分析源码，不难发现，主要是通过循环解析xml文件并将信息解析到内存View对象，布局文件中定义的一个个组件都被顺序的解析到了内存中并被父子View的形式组织起来，这样通过给定的一个root View就可以将整个布局文件中定义的组件全部解析。分析完解析布局文件，回到我们的setContentVIew方法，在调用了installDector方法之后，又调用了：
    ```
    mLayoutInflater.inflate(layoutResID, mContentParent);
    ```
- 这个方法的含义就是将我们传递的客户端的layoutId对应的布局文件作为mContentParent的子View加载到内存中，这样我们的layoutId作为mContentParent的子View，而mContentParent又是mContentRoot的子View，mContentRoot又是mDector的子View，通过LayoutInflater的inflate方法逐步加载到了内存中，而我们的Activity又持有自身的PhoneWindow的引用，这就相当于我们的Activity持有了我们定义的布局文件的引用，因而Activity的布局文件被加载到了内存中。


### 07.关于一点总结
- 总结：
    - Activity的展示界面的特性是通过Window对象来控制的；
    - 每个Activity对象都对应这个一个Window对象，并且Window对象的初始化在启动Activity的时候完成，在执行Activity的onCreate方法之前；
    - 每个Window对象内部都存在一个FrameLayout类型的mDector对象，它是Acitivty界面的root view；
    - Activity中的window对象的实例是PhoneWindow对象，PhoneWindow对象中的几个成员变量mDector，mContentRoot，mContentParent都是View组件，它们的关系是：mDector --> mContentRoot --> mContentParent --> 自定义layoutView
    - LayoutInflater.inflate主要用于将布局文件加载到内存View组件中，也可以设定加载到某一个父组件中；
    - 典型的Activity的onCreate方法中需要调用super.onCreate方法和setContentView方法，若不调用super.onCreate方法，执行启动该Activity的逻辑会报错，若不执行setContentView的方法，该Activity只会显示一个空页面。

## Activity布局绘制

### 01.Activity布局加载简介
- Activity是通过Window来控制界面的展示的，一个Window对象就是一个窗口对象，而每个Activity中都有一个相应的Window对象，所以说一个Activity对象也就可以说是一个窗口对象，而Window只是控制着界面布局文件的加载过程，那么界面布局文件的绘制流程是如何的呢？


### 02.handleResumeActivity
- Android体系在执行Activity的onResume方法之前会回调ActivityThread的handleResumeActivity方法：
    ```
    final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume) {
        ...
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            }
    
        // If the window has already been added, but during resume
        // we started another activity, then don't yet make the
        // window visible.
        }
        ...
        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        if (!r.activity.mFinished && willBeVisible
                && r.activity.mDecor != null && !r.hideForNow) {
            if (r.newConfig != null) {
                r.tmpConfig.setTo(r.newConfig);
                if (r.overrideConfig != null) {
                    r.tmpConfig.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity "
                        + r.activityInfo.name + " with newConfig " + r.tmpConfig);
                performConfigurationChanged(r.activity, r.tmpConfig);
                freeTextLayoutCachesIfNeeded(r.activity.mCurrentConfig.diff(r.tmpConfig));
                r.newConfig = null;
            }
            if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward="
                    + isForward);
            WindowManager.LayoutParams l = r.window.getAttributes();
            if ((l.softInputMode
                    & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                    != forwardBit) {
                l.softInputMode = (l.softInputMode
                        & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                        | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm = a.getWindowManager();
                    View decor = r.window.getDecorView();
                    wm.updateViewLayout(decor, l);
                }
            }
            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
        }
    
        if (!r.onlyLocalRequest) {
            r.nextIdle = mNewActivities;
            mNewActivities = r;
            if (localLOGV) Slog.v(
                TAG, "Scheduling idle handler for " + r);
            Looper.myQueue().addIdleHandler(new Idler());
        }
        r.onlyLocalRequest = false;
    
        // Tell the activity manager we have resumed.
        if (reallyResume) {
            try {
                ActivityManagerNative.getDefault().activityResumed(token);
            } catch (RemoteException ex) {
            }
        }
        ...
    }
    ```
    - 可以看到在在获取了Activity的Window相关参数之后执行了r.activity.makeVisible()方法，看样子这个就是Activity的显示方法
- 这里我们来具体看一下makeVisible方法的具体实现逻辑：
    ```
    void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
    ```
    - 首先判断成员变量mWindowAdded是否为true，可以发现mWindowAdded成员变量只有在执行之后才能赋值为true，所以这里的代码的主要逻辑是该if分支只能执行一次。
    - 这里的ViewManager对象是通过getWindowManager()方法获取的。
- 这个方法主要作用
    - 是将mDecor给显示到界面上


### 03.WindowManager作用
#### 3.1 mWindowManager属性
- 接着上面进行分析。
- 来看一下getWindowManager()方法的具体实现:
    ```
    public WindowManager getWindowManager() {
        return mWindowManager;
    }
    ```
    - 好吧，原来就是返回的Activity的mWindowManager的成员变量，那么这个mWindowManager的成员变量是什么时候赋值的呢？在Activity的attach方法方法中初始化了Activity的相关成员变量，这里也包括了mWindowManager，我们来看一下mWindowManager的赋值过程：

    ```
    mWindowManager = mWindow.getWindowManager();
    ```
- 那么这里的Window对象的mWindowManager成员变量是具体如何赋值的？
    ```
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated
                || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
    ```
    - 可以发现mWindowManager=((WindowManagerImpl)vm).createLocalWindowManager(this)原来是在这里赋值的，所以一个Activity对应这一个新的Window，而这个Window对象内部会对应着一个新的WindowManager对象。
- 接着往下看，那么createLoclWindowManager方法是如何实现的呢？
    ```
    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mDisplay, parentWindow);
    }
    ```
    - 原来是new出了一个WindowManagerImpl对象，所以回到我们的Activity的makeVisible方法，ViewManager获取的是一个WindowManagerImpl对象，所以Window对象内部的WindowManager对象其实都是一个WindowManagerImpl的实例，都是而且从继承关系上可以看到：WindowManagerImpl --> WindowManager --> ViewManager;


#### 3.2 WindowManagerImpl介绍
- 继续往下看：
    ```
    wm.addView(mDecor, getWindow().getAttributes());
    ```
    - 这里的mDector成员变量，我们知道，它是Activity的界面根View，而getWindow.getAttrbutes方法是windowManager中定义的Params内部类，该内部类定义了许多的Window类型，由于这里的vm是WindowManagerImpl的实例。
- 我们来看一下这里的addView的具体实现：
    ```
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }
    ```
- 然后我们具体看一下mGlobal.addView方法，这里的mGlobal是一个WindowManagerGlobal的单例对象，WindowManagerGlobal是Window处理的工具类，那么WindowManagerGlobal的addView具体是如何实现的呢?
    ```
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ...
        ViewRootImpl root;
        View panelParentView = null;
    
        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }
    
            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }
    
            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }
    
            root = new ViewRootImpl(view.getContext(), display);
    
            view.setLayoutParams(wparams);
    
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }
    
        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }
    ```
- 可以发现在WindowManagerGlobal中存在着三个数据列表：
    ```
    private final ArrayList<View> mViews = new ArrayList<View>();
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    private final ArrayList<WindowManager.LayoutParams> mParams =
                new ArrayList<WindowManager.LayoutParams>();
    ```
    - 其中mViews主要用于保存Activity的mDector也就是Activity的根View，
    - 而mRoots主要用于保存ViewRootImpl，
    - mParams主要用于保存Window的LayoutParams，WindowManagerGlobal主要作为WindowManagerImpl的辅助方法类，用于操作View组件。
- 最后我们调用了root.setView方法，这个方法很重要我们就是在这里实现了我们的root与ViewRootImpl的关联的，除了实现了mDector与ViewRootImpl的相互关联，我们还调用了requestLayout方法，这里我们看一下setView方法的具体实现：
    ```
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        ...
        requestLayout();
        ...
    }
    ```
- 可以看到，在方法体中又调用了requestLayout方法，这个方法其实就是调用执行重绘的请求，我们来看一下这个requestLayout方法具体实现：
    ```
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
    ```
- 可以看到这里有一个checkThread方法，这个方法是检查当前线程的方法，若当前线程非UI线程，则抛出非UI线程更新UI的错误：
    ```
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
    ```
    - 相信大家平时在编程的过程中肯定会遇到过这个错误，ViewRootImpl是具体更新View的管理类，所有关于View的更新操作都是在这里执行的，自然而然的对于更新线程的检测是在这个类中添加的，一般在更新UI的时候都会调用这个方法用于检测当前执行更新UI的线程是否是UI线程，否则就会抛出这个异常。


### 04.requestLayout介绍
#### 4.1 requestLayout方法
- 继续回到我们的requestLayout方法，这里又调用了scheduleTraversales方法，我们来看一下这个方法的具体实现：
    ```
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
    ```

#### 4.2 scheduleTraversals()
- 然后看scheduleTraversales方法
    ```
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
    ```
    - 这里mChoreographer.postCallback，内部会调用一个异步消息，用于执行mTraversalRunnable的run方法，这个mTraversalRunnable是一个Runnable对象
- 我们来看一下mTraversalRunnable类的定义：
    ```
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    ```
- 在TraversalRunnable类的run方法中调用了doTraversal方法，我们来看一下这个方法的具体实现逻辑：
    ```
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
    
            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }
    
            performTraversals();
    
            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
    ```
    - 好吧，其内部又回调了方法performTraversals方法


### 05.performTraversals()方法
- 这个方法就是整个View的绘制起始方法，从这个方法开始我们的View经过大小测量，位置测量，界面绘制三个逻辑操作之后就可以展示在界面中了。
    ```
    private void performTraversals() {
        ...
        // 执行View组件的onMeasure方法，主要用于测量View
            if (!mStopped || mReportNextDraw) {
                boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                        (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
                if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                        || mHeight != host.getMeasuredHeight() || contentInsetsChanged) {
                    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    
                    if (DEBUG_LAYOUT) Log.v(TAG, "Ooops, something changed!  mWidth="
                            + mWidth + " measuredWidth=" + host.getMeasuredWidth()
                            + " mHeight=" + mHeight
                            + " measuredHeight=" + host.getMeasuredHeight()
                            + " coveredInsetsChanged=" + contentInsetsChanged);
    
                     // Ask host how big it wants to be
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    
                    // Implementation of weights from WindowManager.LayoutParams
                    // We just grow the dimensions as needed and re-measure if
                    // needs be
                    int width = host.getMeasuredWidth();
                    int height = host.getMeasuredHeight();
                    boolean measureAgain = false;
    
                    if (lp.horizontalWeight > 0.0f) {
                        width += (int) ((mWidth - width) * lp.horizontalWeight);
                        childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                                MeasureSpec.EXACTLY);
                        measureAgain = true;
                    }
                    if (lp.verticalWeight > 0.0f) {
                        height += (int) ((mHeight - height) * lp.verticalWeight);
                        childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                                MeasureSpec.EXACTLY);
                        measureAgain = true;
                    }
    
                    if (measureAgain) {
                        if (DEBUG_LAYOUT) Log.v(TAG,
                                "And hey let's measure once more: width=" + width
                                + " height=" + height);
                        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                    }
    
                    layoutRequested = true;
                }
            }
        }
    	...
    	// 主要用于测量View组件的位置
    	...
        final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
        boolean triggerGlobalLayoutListener = didLayout
                || mAttachInfo.mRecomputeGlobalAttributes;
        if (didLayout) {
            performLayout(lp, desiredWindowWidth, desiredWindowHeight);
    
            // By this point all views have been sized and positioned
            // We can compute the transparent area
    
            if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
                // start out transparent
                // TODO: AVOID THAT CALL BY CACHING THE RESULT?
                host.getLocationInWindow(mTmpLocation);
                mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
                        mTmpLocation[0] + host.mRight - host.mLeft,
                        mTmpLocation[1] + host.mBottom - host.mTop);
    
                host.gatherTransparentRegion(mTransparentRegion);
                if (mTranslator != null) {
                    mTranslator.translateRegionInWindowToScreen(mTransparentRegion);
                }
    
                if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {
                    mPreviousTransparentRegion.set(mTransparentRegion);
                    mFullRedrawNeeded = true;
                    // reconfigure window manager
                    try {
                        mWindowSession.setTransparentRegion(mWindow, mTransparentRegion);
                    } catch (RemoteException e) {
                    }
                }
            }
    
            if (DBG) {
                System.out.println("======================================");
                System.out.println("performTraversals -- after setFrame");
                host.debug();
            }
        }
        ...
        // 主要用于View的绘制过程
        ...
        if (!cancelDraw && !newSurface) {
            if (!skipDraw || mReportNextDraw) {
                if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                    for (int i = 0; i < mPendingTransitions.size(); ++i) {
                        mPendingTransitions.get(i).startChangingAnimations();
                    }
                    mPendingTransitions.clear();
                }
    
                performDraw();
            }
        } else {
            if (viewVisibility == View.VISIBLE) {
                // Try again
                scheduleTraversals();
            } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).endChangingAnimations();
                }
                mPendingTransitions.clear();
            }
        }
    
        mIsInTraversal = false;
    }
    ```
- 可以看到在方法performTraversals方法，我们调用了performMeasure，performLayout，performDraw三个方法，这几个方法主要用于测量View组件的大小，测量View组件的位置，绘制View组件；
    - 即：测量大小 --> 测量位置 --> 绘制组件


#### 5.1 performMeasure测量
- 好吧，这里我们调用了performMeasure方法，我们先看一下performMeasure方法的具体实现：
    ```
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
    ```
- 可以看到在performMeasure方法中我们又调用了mView的measure方法，这里的mView就是我们一开始的Activity的mDector根组件，这里的measure方法就是调用的mDector组件的measure方法：
    ```
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        ...
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        ...
    }
    ```
- 在View的measure方法中，又调用了onMeasure方法，由于我们的mDector对象是一个FrameLayout，所以这里的onMeasure执行的是FrameLayout的onMeasure方法：
    ```
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();
    
        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();
    
        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;
    
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }
    
        // Account for padding too
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();
    
        // Check against our minimum height and width
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());
    
        // Check against our foreground's minimum height and width
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }
    
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));
    
        count = mMatchParentChildren.size();
        if (count > 1) {
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);
                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    
                final int childWidthMeasureSpec;
                if (lp.width == LayoutParams.MATCH_PARENT) {
                    final int width = Math.max(0, getMeasuredWidth()
                            - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                            - lp.leftMargin - lp.rightMargin);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            width, MeasureSpec.EXACTLY);
                } else {
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }
    
                final int childHeightMeasureSpec;
                if (lp.height == LayoutParams.MATCH_PARENT) {
                    final int height = Math.max(0, getMeasuredHeight()
                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                            - lp.topMargin - lp.bottomMargin);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            height, MeasureSpec.EXACTLY);
                } else {
                    childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                            getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                            lp.topMargin + lp.bottomMargin,
                            lp.height);
                }
    
                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }
    }
    ```
- 可以看到这里调用了一个循环逻辑，获取该View的所有子View，并执行所有子View的measure方法，这样又回到View的measure方法，这样经过一系列的循环遍历过程，如果是ViewGroup就会调用其ViewGroup的onMeasure方法，若果是View组件就会调用View的onMeasure方法，我们来看一下View的onMeasure方法：
    ```
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    ```
- 可以看到这个方法中调用了setMeasuredDimension方法：
    ```
    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;
    
            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }
    ```
- 好吧，方法体里面又调用了setMeasuredDimensionRaw方法：
    ```
    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;
    
        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
    ```

- 这样把View组件即其子View的大小测量出来了，并且保存在了成员变量mMeasuredWith和mMeasuredHeight中。



#### 5.2 performLayout布局
- 继续回到我们的performTransles方法，然后我们继续看performLayout方法：
    ```
    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;
    
        final View host = mView;
        if (DEBUG_ORIENTATION || DEBUG_LAYOUT) {
            Log.v(TAG, "Laying out " + host + " to (" +
                    host.getMeasuredWidth() + ", " + host.getMeasuredHeight() + ")");
        }
    
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
        try {
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    
            mInLayout = false;
            int numViewsRequestingLayout = mLayoutRequesters.size();
            if (numViewsRequestingLayout > 0) {
                // requestLayout() was called during layout.
                // If no layout-request flags are set on the requesting views, there is no problem.
                // If some requests are still pending, then we need to clear those flags and do
                // a full request/measure/layout pass to handle this situation.
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                        false);
                if (validLayoutRequesters != null) {
                    // Set this flag to indicate that any further requests are happening during
                    // the second pass, which may result in posting those requests to the next
                    // frame instead
                    mHandlingLayoutInLayoutRequest = true;
    
                    // Process fresh layout requests, then measure and layout
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        Log.w("View", "requestLayout() improperly called by " + view +
                                " during layout: running second layout pass");
                        view.requestLayout();
                    }
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);
                    mInLayout = true;
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    
                    mHandlingLayoutInLayoutRequest = false;
    
                    // Check the valid requests again, this time without checking/clearing the
                    // layout flags, since requests happening during the second pass get noop'd
                    validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                    if (validLayoutRequesters != null) {
                        final ArrayList<View> finalRequesters = validLayoutRequesters;
                        // Post second-pass requests to the next frame
                        getRunQueue().post(new Runnable() {
                            @Override
                            public void run() {
                                int numValidRequests = finalRequesters.size();
                                for (int i = 0; i < numValidRequests; ++i) {
                                    final View view = finalRequesters.get(i);
                                    Log.w("View", "requestLayout() improperly called by " + view +
                                            " during second layout pass: posting in next frame");
                                    view.requestLayout();
                                }
                            }
                        });
                    }
                }
    
            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }
    ```
- 可以看到在方法体中，我们看到该方法执行了layout方法，我们看一下该layout方法的实现：
    ```
    public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
    
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
    
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
    
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }
    
        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }
    ```
- 可以看到这个方法体中执行了onLayout方法，这个方法就是具体执行测量位置的方法了，由于我们的mDector是一个FrameLayout，所以跟measure类似的，我们看一下FrameLayout的onLayout方法的实现：
    ```
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }
    ```
    - 可以看到这里调用了layoutChildren方法，让我们来看一下layoutChildren方法的实现：
    ```
    void layoutChildren(int left, int top, int right, int bottom,
                                  boolean forceLeftGravity) {
        final int count = getChildCount();
    
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();
    
        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();
    
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    
                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();
    
                int childLeft;
                int childTop;
    
                int gravity = lp.gravity;
                if (gravity == -1) {
                    gravity = DEFAULT_CHILD_GRAVITY;
                }
    
                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;
    
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT:
                        if (!forceLeftGravity) {
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    case Gravity.LEFT:
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }
    
                switch (verticalGravity) {
                    case Gravity.TOP:
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL:
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM:
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:
                        childTop = parentTop + lp.topMargin;
                }
    
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
    ```
- 跟measure类似的，这里也是遍历执行View的layout方法，若是ViewGroup则执行具体的ViewGroup的layout方法，若是View，则执行View的layout方法，好吧，我们看一下View的layout的具体实现逻辑：
    ```
    public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
    
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
    
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
    
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }
    
        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }
    ```
    - 这样经过layout方法，如果是View组件的话就已经将View组件的位置信息计算出来并保存在对象的成员变量中。



### 5.3 performDraw绘制
- 经过了测量大小与测量位置的逻辑之后，最后看一下performTraversals方法中的performDraw方法，这个方法的作用就是执行View组件的绘制逻辑了。
    ```
    private void performDraw() {
        draw(fullRedrawNeeded);
    }
    ```
- 可以看到这里调用了ViewRootImpl的draw方法，然后我们看一下draw方法的实现：
    ```
    private void draw(boolean fullRedrawNeeded) {
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
           return;
        }
    }
    ```
- 可以看到这里又调用了drawSoftware方法，看名字这里应该就是调用执行绘制的方法：
    ```
    private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {
        mView.draw(canvas);
        return true;
    }
    ```
- 可以看到这里调用了mView的draw方法，这里的mView是我们的mDector，好吧，看一下draw方法的具体实现：
    ```
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
    
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */
    
        // Step 1, draw the background, if needed
        int saveCount;
    
        if (!dirtyOpaque) {
            drawBackground(canvas);
        }
    
        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);
    
            // Step 4, draw the children
            dispatchDraw(canvas);
    
            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }
    
            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);
    
            // we're done...
            return;
        }
    
        /*
         * Here we do the full fledged routine...
         * (this is an uncommon case where speed matters less,
         * this is why we repeat some of the tests that have been
         * done above)
         */
    
        boolean drawTop = false;
        boolean drawBottom = false;
        boolean drawLeft = false;
        boolean drawRight = false;
    
        float topFadeStrength = 0.0f;
        float bottomFadeStrength = 0.0f;
        float leftFadeStrength = 0.0f;
        float rightFadeStrength = 0.0f;
    
        // Step 2, save the canvas' layers
        int paddingLeft = mPaddingLeft;
    
        final boolean offsetRequired = isPaddingOffsetRequired();
        if (offsetRequired) {
            paddingLeft += getLeftPaddingOffset();
        }
    
        int left = mScrollX + paddingLeft;
        int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
        int top = mScrollY + getFadeTop(offsetRequired);
        int bottom = top + getFadeHeight(offsetRequired);
    
        if (offsetRequired) {
            right += getRightPaddingOffset();
            bottom += getBottomPaddingOffset();
        }
    
        final ScrollabilityCache scrollabilityCache = mScrollCache;
        final float fadeHeight = scrollabilityCache.fadingEdgeLength;
        int length = (int) fadeHeight;
    
        // clip the fade length if top and bottom fades overlap
        // overlapping fades produce odd-looking artifacts
        if (verticalEdges && (top + length > bottom - length)) {
            length = (bottom - top) / 2;
        }
    
        // also clip horizontal fades if necessary
        if (horizontalEdges && (left + length > right - length)) {
            length = (right - left) / 2;
        }
    
        if (verticalEdges) {
            topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
            drawTop = topFadeStrength * fadeHeight > 1.0f;
            bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
            drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
        }
    
        if (horizontalEdges) {
            leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
            drawLeft = leftFadeStrength * fadeHeight > 1.0f;
            rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
            drawRight = rightFadeStrength * fadeHeight > 1.0f;
        }
    
        saveCount = canvas.getSaveCount();
    
        int solidColor = getSolidColor();
        if (solidColor == 0) {
            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;
    
            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }
    
            if (drawBottom) {
                canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
            }
    
            if (drawLeft) {
                canvas.saveLayer(left, top, left + length, bottom, null, flags);
            }
    
            if (drawRight) {
                canvas.saveLayer(right - length, top, right, bottom, null, flags);
            }
        } else {
            scrollabilityCache.setFadeColor(solidColor);
        }
    
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);
    
        // Step 4, draw the children
        dispatchDraw(canvas);
    
        // Step 5, draw the fade effect and restore layers
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;
    
        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }
    
        if (drawBottom) {
            matrix.setScale(1, fadeHeight * bottomFadeStrength);
            matrix.postRotate(180);
            matrix.postTranslate(left, bottom);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, bottom - length, right, bottom, p);
        }
    
        if (drawLeft) {
            matrix.setScale(1, fadeHeight * leftFadeStrength);
            matrix.postRotate(-90);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, left + length, bottom, p);
        }
    
        if (drawRight) {
            matrix.setScale(1, fadeHeight * rightFadeStrength);
            matrix.postRotate(90);
            matrix.postTranslate(right, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(right - length, top, right, bottom, p);
        }
    
        canvas.restoreToCount(saveCount);
    
        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }
    
        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);
    }
    ```
- 整个View的绘制流程还是比较清楚的，整个执行逻辑还有相应的注释，并且在执行draw方法的过程中，如果包含子View，那么也会执行子View的draw方法，好吧，经过这样一系列的执行逻辑之后，mDector以及子View就被绘制出来了。



### 06.Activity布局绘制总结
- 总结如下所示：
    - Activity执行onResume之后再ActivityThread中执行Activity的makeVisible方法。
    - View的绘制流程包含了测量大小，测量位置，绘制三个流程；
    - Activity的界面绘制是从mDoctor即根View开始的，也就是从mDoctor的测量大小，测量位置，绘制三个流程；
    - View体系的绘制流程是从ViewRootImpl的performTraversals方法开始的；
    - View的测量大小流程:performMeasure --> measure --> onMeasure等方法;
    - View的测量位置流程：performLayout --> layout --> onLayout等方法；
    - View的绘制流程：onDraw等方法；
    - View组件的绘制流程会在onMeasure,onLayout以及onDraw方法中执行分发逻辑，也就是在onMeasure同时执行子View的测量大小逻辑，在onLayout中同时执行子View的测量位置逻辑，在onDraw中同时执行子View的绘制逻辑；
    - Activity中都对应这个一个Window对象，而每一个Window对象都对应着一个新的WindowManager对象（WindowManagerImpl实例）；



### 07.Activity布局绘制流程图
- 如图所示
    - ![img](http://upload-images.jianshu.io/upload_images/3985563-5f3c64af676d9aee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- View的绘制是从上往下一层层迭代下来的。DecorView-->ViewGroup（--->ViewGroup）-->View ，按照这个流程从上往下，依次measure(测量),layout(布局),draw(绘制)。
    - ![img](http://upload-images.jianshu.io/upload_images/3985563-a7ace6f9221c9d79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Android 构建流程
